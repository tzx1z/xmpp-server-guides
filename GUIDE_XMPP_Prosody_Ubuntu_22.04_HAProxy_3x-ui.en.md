# Step-by-step guide: installing the Prosody XMPP server on a VPS in Russia (Ubuntu 22.04)

Versions: Ubuntu 22.04.5 LTS, Prosody 13.0.7 from the packages.prosody.im repository, coturn 4.5.2 and certbot 1.21 with the dns-cloudflare plugin from the Ubuntu repository. The domain's DNS is hosted on Cloudflare. The Prosody and coturn configurations in this guide were tested on these versions.

## Contents

1. [Introduction](#1-introduction)
2. [Checking the current state of the VPS](#2-checking-the-current-state-of-the-vps)
3. [DNS records in Cloudflare](#3-dns-records-in-cloudflare)
4. [Preparing the system and installing Prosody](#4-preparing-the-system-and-installing-prosody)
5. [Firewall](#5-firewall)
6. [SSL certificate](#6-ssl-certificate)
7. [Prosody configuration](#7-prosody-configuration)
8. [Calls: coturn](#8-calls-coturn)
9. [Startup and verification](#9-startup-and-verification)
10. [Users and clients](#10-users-and-clients)
11. [Security and maintenance](#11-security-and-maintenance)
12. [Troubleshooting](#12-troubleshooting)
13. [Summary](#13-summary)
14. [Appendix A. XMPP and file sharing over port 443](#appendix-a-xmpp-and-file-sharing-over-port-443)

---

## 1. Introduction

### 1.1. What XMPP and Prosody are

XMPP is an open messaging protocol. XMPP servers talk to each other the same way mail servers do: a user on the `inkov.dev` server can chat with users of any other XMPP server. Prosody is an XMPP server written in Lua. It uses tens of megabytes of memory, is configured with a single file and suits a personal server. After you complete this guide, the server handles addresses like `user@inkov.dev`, keeps message history, transfers files, and supports group chats and calls.

### 1.2. Layout

```text
Cloudflare DNS, zone inkov.dev
├── inkov.dev                    A      GitHub Pages (unchanged)
├── xmpp.inkov.dev               A      <IP_VPS>
├── conference.inkov.dev         CNAME  xmpp.inkov.dev
├── upload.inkov.dev             CNAME  xmpp.inkov.dev
├── turn.inkov.dev               CNAME  xmpp.inkov.dev
├── _xmpp-client._tcp.inkov.dev  SRV    0 5 5222 xmpp.inkov.dev
└── _xmpp-server._tcp.inkov.dev  SRV    0 5 5269 xmpp.inkov.dev

VPS <IP_VPS>, Ubuntu 22.04
├── 3x-ui / Xray   443/tcp, panel port, inbound ports   (unchanged)
├── Prosody        5222/tcp          client connections
│                  5269/tcp          server-to-server connections
│                  5281/tcp          HTTPS: file sharing
│                  5280/tcp          127.0.0.1 only
└── coturn         3478/tcp and udp  STUN/TURN for calls
                   50000-50100/udp   media relay
```

How a connection is made:

1. A client with the address `user@inkov.dev` looks up the SRV record `_xmpp-client._tcp.inkov.dev` and gets back server `xmpp.inkov.dev`, port 5222.
2. The client connects to the VPS, and Prosody presents a certificate for `inkov.dev`.
3. Other XMPP servers find the server through the SRV record `_xmpp-server._tcp.inkov.dev`, port 5269.
4. The GitHub Pages site keeps working: the SRV records have different names and do not overlap with the `inkov.dev` A records.

### 1.3. Design decisions

- **Addresses are `user@inkov.dev`, the server runs on `xmpp.inkov.dev`.** Prosody is configured with `VirtualHost "inkov.dev"`, and SRV records point the domain to the server.
- **Certificate for `inkov.dev` and `*.inkov.dev`.** Clients and other servers check the certificate against the domain in the address (RFC 6120), not against the name from the SRV record. The apex domain `inkov.dev` points to GitHub, so the HTTP challenge (HTTP-01) cannot be passed from the VPS. The DNS challenge (DNS-01) is used instead: certbot creates a temporary TXT record through the Cloudflare API. One wildcard certificate covers `inkov.dev` and all subdomains.
- **Prosody 13 from the official repository.** Ubuntu 22.04 ships version 0.11, which has neither file sharing (`mod_http_file_share`) nor support for an external TURN server (`mod_turn_external`).
- **Port 443 is not used in the main setup.** Xray holds it, so file sharing runs over HTTPS on port 5281. Appendix A describes how to move client connections and file sharing to 443 without decrypting traffic.
- **3x-ui is left as is.** The steps on the VPS only add packages, configuration files and firewall rules.
- **Calls go through coturn.** `mod_turn_external` only hands clients the address and temporary credentials of the TURN server. The TURN server itself is installed separately (section 8).

### 1.4. Limitations

- Ports 5222 and 5281 are non-standard. On networks that allow only port 443 (some corporate networks), connecting and file sharing will not work. Appendix A covers such networks.
- The VPS IP address is published in DNS as the address of `xmpp.inkov.dev`. If 3x-ui disguises its traffic as a third-party site (for example, VLESS Reality), open XMPP ports and an `inkov.dev` certificate on the same IP make it possible to link the server to the domain.
- The domain in user addresses can only be changed together with the accounts: after a domain change they have to be created again.

### 1.5. Parameters

| Parameter | Value | Used in |
|---|---|---|
| XMPP domain | `inkov.dev` | user addresses, `VirtualHost` |
| Server host | `xmpp.inkov.dev` | A record, SRV target |
| `<IP_VPS>` | public IPv4 of the VPS, `203.0.113.10` in the examples | DNS, checks |
| `<EMAIL>` | email for the Let's Encrypt account | certbot |
| `<CF_API_TOKEN>` | Cloudflare API token | only the file `/root/.secrets/cloudflare.ini` |
| `<SSH_PORT>` | SSH port, usually 22 | firewall |
| `<PANEL_PORT>` | 3x-ui panel port | firewall |
| `<TURN_SECRET>` | shared secret for Prosody and coturn | created in section 7.2, inserted by a command |

Commands in code blocks run on the VPS unless stated otherwise. Replace the placeholders in angle brackets with your own values.

### 1.6. What you need

- A VPS with Ubuntu 22.04, SSH access, a user with sudo rights.
- Access to the Cloudflare dashboard for `inkov.dev`.
- Access to the VPS console in the provider's control panel (VNC or similar) in case you make a mistake in the firewall setup.
- An XMPP client on a phone or computer (section 10).

---

## 2. Checking the current state of the VPS

This section collects the data needed for the setup and checks that the XMPP ports are free. Its commands only read the system state, except for the backup in 2.6.

### 2.1. OS and resources

```bash
# OS version: Ubuntu 22.04.x LTS expected
lsb_release -ds
# Free space on the root partition
df -h /
# Free memory
free -h
```

**Expected:** `Ubuntu 22.04.5 LTS`, at least 6 GB of free disk space: up to 5 GB is reserved for user files (section 7). If you have less space, lower `http_file_share_global_quota` in the Prosody configuration. Prosody and coturn together use less than 100 MB of memory.

### 2.2. Time

```bash
timedatectl
```

**Expected:** `System clock synchronized: yes` and `NTP service: active`. A wrong clock causes certificate validation errors. If synchronization is off:

```bash
sudo timedatectl set-ntp true
```

On a VPS with OpenVZ or LXC virtualization the clock is managed by the provider: if it drifts, contact support.

### 2.3. Ports in use and 3x-ui ports

```bash
# All listening ports with process names
sudo ss -tulpn
# 3x-ui and Xray ports: needed in section 5
sudo ss -tulpn | grep -E 'x-ui|xray'
# SSH port
sudo ss -tlpn | grep sshd
```

Example output of the second command:

```text
tcp   LISTEN 0  4096   *:443    *:*   users:(("xray-linux-amd6",pid=812,fd=7))
tcp   LISTEN 0  4096   *:2053   *:*   users:(("x-ui",pid=790,fd=9))
```

Here 443 is the Xray inbound and 2053 is the 3x-ui panel. Write down all ports from the output together with the protocol (`tcp` or `udp`).

Check that the Prosody and coturn ports are free:

```bash
sudo ss -tulpn | grep -E ':(5222|5269|5280|5281|5000|3478|5349)\b'
sudo ss -tulpn | grep -E ':50(0[0-9]{2}|100)\b'
```

**Expected:** empty output. If a port is taken by a 3x-ui process, leave 3x-ui alone and change the port in this guide instead: port 5281 and the coturn range are changed only in the configuration, ports 5222 and 5269 in the configuration and in the SRV records.

### 2.4. Firewall and the 3x-ui panel certificate

```bash
sudo ufw status verbose
```

- `Status: active` - ufw is on, section 5 only adds rules.
- `Status: inactive` - ufw is off, section 5 describes how to turn it on safely.

If ufw is inactive, check for other filtering rules:

```bash
sudo iptables -S | head -20
```

The lines `-P INPUT ACCEPT`, `-P FORWARD ACCEPT`, `-P OUTPUT ACCEPT` with no other rules mean there is no filtering.

Check whether the 3x-ui panel certificate was obtained through the `x-ui` menu:

```bash
sudo ls /root/.acme.sh 2>/dev/null
```

If the output contains directories named after domains, the panel certificate is renewed by acme.sh. If the certificate was issued in standalone mode, acme.sh uses port 80 for renewal, and this port has to stay open (section 5).

Also check the provider's control panel for an external (cloud) firewall. If it is enabled, the ports from section 5 must be opened there too.

### 2.5. IP addresses and IPv6

```bash
# Addresses on network interfaces
ip -br addr
# Public IPv4 as seen by external servers
curl -4 -s https://ipv4-internet.yandex.net/api/v0/ip; echo
# Public IPv6: an error or empty response means IPv6 does not work
curl -6 -s --max-time 5 https://ipv6-internet.yandex.net/api/v0/ip; echo
```

- The public IPv4 from the second command shows up in `ip -br addr` - the VPS gets the address directly. This is the usual case.
- The interface has only a private address (`10.x.x.x`, `172.16-31.x.x`, `192.168.x.x`) - the provider uses NAT. coturn then needs the `external-ip` option (section 8.2).
- The third command returned an address - IPv6 works, and you can create an AAAA record for `xmpp.inkov.dev`. If there is no address, do not create an AAAA record.

### 2.6. Backup of the 3x-ui database

This guide does not change 3x-ui. The backup is there in case something goes wrong while you configure the firewall or the system.

```bash
sudo cp /etc/x-ui/x-ui.db /root/x-ui.db.bak-$(date +%F)
sudo ls -l /root/x-ui.db.bak-*
```

---

## 3. DNS records in Cloudflare

Run on: the Cloudflare dashboard in a browser. The check runs on the VPS or on your local computer.

### 3.1. Creating the records

1. Open https://dash.cloudflare.com and select the `inkov.dev` domain.
2. Go to **DNS → Records**.
3. For each row in the table click **Add record**, fill in the fields and click **Save**.

| Type | Name | Value | Proxy status | TTL |
|---|---|---|---|---|
| A | `xmpp` | IPv4 address: `<IP_VPS>` | DNS only | Auto |
| CNAME | `conference` | Target: `xmpp.inkov.dev` | DNS only | Auto |
| CNAME | `upload` | Target: `xmpp.inkov.dev` | DNS only | Auto |
| CNAME | `turn` | Target: `xmpp.inkov.dev` | DNS only | Auto |
| SRV | `_xmpp-client._tcp` | Priority `0`, Weight `5`, Port `5222`, Target `xmpp.inkov.dev` | n/a | Auto |
| SRV | `_xmpp-server._tcp` | Priority `0`, Weight `5`, Port `5269`, Target `xmpp.inkov.dev` | n/a | Auto |
| AAAA, only if IPv6 works | `xmpp` | IPv6 address: the VPS address | DNS only | Auto |
| CNAME, only for Proxy65 | `proxy` | Target: `xmpp.inkov.dev` | DNS only | Auto |

> **Important:** for A, AAAA and CNAME records Cloudflare turns on proxying by default (Proxy status: Proxied, orange cloud). Switch it to **DNS only** (grey cloud). The Cloudflare proxy passes only HTTP and HTTPS on a limited set of ports, so XMPP and TURN do not work through it.

Notes on the fields:

- **Name** is relative to the zone: `xmpp` means `xmpp.inkov.dev`, `_xmpp-client._tcp` means `_xmpp-client._tcp.inkov.dev`. Cloudflare appends the zone name itself.
- The SRV records go into the `inkov.dev` zone, not a subdomain: clients look them up by the domain from the user's address. If the SRV form shows separate Service and Protocol fields, set Service to `_xmpp-client` (or `_xmpp-server`), Protocol to `TCP`, Name to `@`.
- The SRV **Target** is a name with an A record (`xmpp.inkov.dev`), not a CNAME: RFC 2782 requires this.
- **Priority** sets the order in which servers are tried, **Weight** spreads the load between servers with the same priority. For a single server any values work; `0` and `5` are a common choice.
- **TTL Auto** in Cloudflare is 300 seconds.
- For DNS only records the dashboard may warn that the IP address will be exposed. For this setup that is expected.
- Do not touch the existing `inkov.dev` A records (GitHub Pages) or the `www` record, if you have one.

### 3.2. CAA records

CAA records restrict which certificate authorities may issue certificates for the domain.

```bash
# If dig is not found: sudo apt install -y dnsutils
dig +short CAA inkov.dev
```

- Empty output - there are no restrictions and nothing else to do.
- If there are records, one of them must be `0 issue "letsencrypt.org"`. If there are records with the `issuewild` tag, you also need `0 issuewild "letsencrypt.org"`, otherwise the wildcard certificate will not be issued. Add the missing record in Cloudflare: Type `CAA`, Name `@`, pick the tag in the **Tag** field, value `letsencrypt.org`.

GitHub Pages also gets its certificates from Let's Encrypt, so existing CAA records usually allow `letsencrypt.org` already.

### 3.3. Verification

Queries sent straight to the Cloudflare DNS server show the records right after you save them, without the cache of intermediate resolvers.

```bash
# Cloudflare DNS server for the zone
NS=$(dig +short NS inkov.dev | head -1); echo "$NS"
dig +short @"$NS" A xmpp.inkov.dev
dig +short @"$NS" SRV _xmpp-client._tcp.inkov.dev
dig +short @"$NS" SRV _xmpp-server._tcp.inkov.dev
dig +short @"$NS" upload.inkov.dev
```

**Expected** (with the example address `203.0.113.10`):

```text
xxxx.ns.cloudflare.com.
203.0.113.10
0 5 5222 xmpp.inkov.dev.
0 5 5269 xmpp.inkov.dev.
xmpp.inkov.dev.
203.0.113.10
```

Then check the answer from a regular resolver:

```bash
dig +short SRV _xmpp-client._tcp.inkov.dev
```

The answer must match the previous one. If it is empty, wait: a resolver that already queried this name before the record existed keeps the negative answer for up to 30 minutes.

---

## 4. Preparing the system and installing Prosody

### 4.1. APT mirrors

```bash
sudo apt update
```

**Expected:** `Hit:` and `Get:` lines, no `Err:` lines and no hangs. In that case leave the mirrors alone: Russian providers often use local ones already.

If `apt update` fails with `Err:` errors or hangs, switch the system to the Yandex mirror:

```bash
# Backup of the current list
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
# New list: main repository, updates, backports and security updates
sudo tee /etc/apt/sources.list > /dev/null <<'EOF'
deb https://mirror.yandex.ru/ubuntu jammy main restricted universe multiverse
deb https://mirror.yandex.ru/ubuntu jammy-updates main restricted universe multiverse
deb https://mirror.yandex.ru/ubuntu jammy-backports main restricted universe multiverse
deb https://mirror.yandex.ru/ubuntu jammy-security main restricted universe multiverse
EOF
sudo apt update
```

Rollback:

```bash
sudo cp /etc/apt/sources.list.bak /etc/apt/sources.list && sudo apt update
```

On Ubuntu 22.04 the main repository list is `/etc/apt/sources.list`. The file `/etc/apt/sources.list.d/ubuntu.sources` only appeared in Ubuntu 24.04. If the error comes from an additional repository, find its file with `ls /etc/apt/sources.list.d/`.

### 4.2. System update

```bash
sudo apt upgrade -y
# Is a reboot needed after a kernel update
ls /var/run/reboot-required 2>/dev/null && echo "reboot required"
```

If a reboot is needed, run `sudo reboot` at a convenient time: the 3x-ui VPN is down while the server reboots. Do not run `do-release-upgrade`: this guide is written for Ubuntu 22.04.

### 4.3. Prosody repository

```bash
# Repository reachability from the VPS: 200 expected
curl -s -o /dev/null -w '%{http_code}\n' --max-time 15 http://packages.prosody.im/debian/dists/jammy/Release
# Add the repository. The file contains the repository URL and the signing key (Signed-By),
# no separate key or apt-key needed
sudo wget https://prosody.im/downloads/repos/jammy/prosody.sources -O /etc/apt/sources.list.d/prosody.sources
sudo apt update
# Version that will be installed
apt-cache policy prosody
```

**Expected output of** `apt-cache policy prosody`:

```text
prosody:
  Installed: (none)
  Candidate: 13.0.7-1~jammy2
  Version table:
     13.0.7-1~jammy2 500
        500 http://packages.prosody.im/debian jammy/main amd64 Packages
     0.11.13-1 500
        500 http://archive.ubuntu.com/ubuntu jammy/universe amd64 Packages
```

The version number may be newer. What matters is that `Candidate` comes from `packages.prosody.im` and is 13.0 or later.

If the repository is unreachable (curl returns `000` or times out), download the `prosody_13.*~jammy*_amd64.deb` package on another computer from https://packages.prosody.im/debian/pool/main/p/prosody/, copy it to the VPS with `scp` and install it with `sudo apt install --no-install-recommends ./prosody_*.deb lua-unbound lua-readline`. Do not install version 0.11 from the Ubuntu repository.

### 4.4. Installing Prosody

```bash
sudo apt install --no-install-recommends prosody lua-unbound lua-readline
```

> **Important:** the `--no-install-recommends` flag is required. Without it apt installs the recommended `luarocks` package. On Ubuntu 22.04 it depends on Lua 5.1 and switches the `lua` command to it, after which Prosody fails to start with `Prosody is no longer compatible with Lua 5.1`. For the same reason do not install `luarocks`, `lua5.4` or `liblua5.4-dev` by hand: the required Lua 5.4 comes in as a package dependency.

`lua-unbound` (DNS resolver) and `lua-readline` (line editing in `prosodyctl shell`) are recommended by Prosody and do not pull in extra dependencies.

Check:

```bash
sudo prosodyctl about | grep -E '^Prosody|Lua version|luaunbound'
systemctl status prosody --no-pager
```

**Expected:**

```text
Prosody 13.0.7
Lua version:             	Lua 5.4
luaunbound:   	1.0.0
```

and `Active: active (running)`. Prosody is now running with the default configuration (host `localhost`), which section 7 replaces.

---

## 5. Firewall

### 5.1. Rules

> **Important:** if ufw is currently inactive, turning it on without rules for SSH and 3x-ui will cut off access to the server and stop the VPN. Add all rules before `ufw enable`. Do not close the current SSH session until you have tested a new connection.

```bash
# 1. SSH and 3x-ui: ports from section 2.3
sudo ufw allow <SSH_PORT>/tcp comment 'SSH'
sudo ufw allow <PANEL_PORT>/tcp comment '3x-ui panel'
sudo ufw allow 443/tcp comment 'Xray'
# Other Xray inbound ports the same way, for UDP: sudo ufw allow <PORT>/udp
# If section 2.4 found the /root/.acme.sh directory:
# sudo ufw allow 80/tcp comment 'acme.sh 3x-ui'

# 2. XMPP
sudo ufw allow 5222/tcp comment 'XMPP clients'
sudo ufw allow 5269/tcp comment 'XMPP servers'
sudo ufw allow 5281/tcp comment 'XMPP HTTPS file share'

# 3. List of added rules
sudo ufw show added
```

**Expected output of** `ufw show added`: a list of all rules, including SSH and the 3x-ui ports. If ufw was already active, the rules take effect at once; go to 5.3.

Port 5280 is not opened: it is only needed locally. The coturn ports (3478 and 50000-50100/udp) are opened in section 8.3, after coturn is configured. Ports 5000 (Proxy65) and 5349 (TURN over TLS) are opened only if you enable those features (sections 7.3 and 8.4).

### 5.2. Enabling ufw (only if it was inactive)

```bash
sudo ufw enable
sudo ufw status verbose
```

`ufw enable` warns that SSH connections may be interrupted; confirm with `y`. Then, without closing the current session, open a new SSH connection and check that the VPN works. If you lose access, run `sudo ufw disable` in the provider's web console.

**Expected:**

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
...
5222/tcp                   ALLOW IN    Anywhere                   # XMPP clients
```

The rules apply to both IPv4 and IPv6: `/etc/default/ufw` has `IPV6=yes` by default.

> **Important:** if, besides 3x-ui, the VPS runs a VPN that routes traffic in the kernel (WireGuard, OpenVPN, AmneziaWG), an enabled ufw blocks forwarding (`DEFAULT_FORWARD_POLICY="DROP"` in `/etc/default/ufw`). In that case set up forwarding separately or leave ufw off. Xray in 3x-ui works as a proxy and does not use forwarding.

### 5.3. Provider firewall

If section 2.4 found an external firewall in the provider's control panel, open the same ports there: 5222/tcp, 5269/tcp, 5281/tcp. Open the coturn ports (3478/tcp and udp, 50000-50100/udp) there as well after section 8.3.

---

## 6. SSL certificate

### 6.1. Let's Encrypt reachability

```bash
curl -s -o /dev/null -w '%{http_code}\n' --max-time 15 https://acme-v02.api.letsencrypt.org/directory
```

**Expected:** `200`. Whether Let's Encrypt is reachable from Russia depends on the provider's network, so the guide relies on this check. If you get `000` or any other code, go to the fallback options in 6.8.

### 6.2. Cloudflare API token

certbot creates the temporary TXT record `_acme-challenge.inkov.dev` through the Cloudflare API. This needs a token that can edit DNS in the `inkov.dev` zone only.

Run on: the Cloudflare dashboard.

1. Open **My Profile → API Tokens** (https://dash.cloudflare.com/profile/api-tokens) and click **Create Token**.
2. Pick the **Edit zone DNS** template and click **Use template**.
3. **Permissions**: leave `Zone` / `DNS` / `Edit`.
4. **Zone Resources**: `Include` / `Specific zone` / `inkov.dev`.
5. **Client IP Address Filtering** (recommended): `Is in` and the address `<IP_VPS>`. If IPv6 works on the VPS, add the IPv6 address too: API requests may go out over IPv6.
6. Click **Continue to summary**, then **Create Token**.
7. Copy the token. Cloudflare shows it only once.

### 6.3. Token on the VPS

```bash
# Directory and empty file accessible only to root
sudo install -d -m 700 /root/.secrets
sudo install -m 600 /dev/null /root/.secrets/cloudflare.ini
sudo nano /root/.secrets/cloudflare.ini
```

File contents:

```ini
# Cloudflare API token: Zone / DNS / Edit, inkov.dev zone only
dns_cloudflare_api_token = <CF_API_TOKEN>
```

The token is pasted in the editor, so it does not end up in the shell history.

Check the token and whether the Cloudflare API is reachable from the VPS (the token is read from the file):

```bash
sudo sh -c 'curl -s --max-time 20 https://api.cloudflare.com/client/v4/user/tokens/verify -H "Authorization: Bearer $(sed -n "s/^dns_cloudflare_api_token *= *//p" /root/.secrets/cloudflare.ini)"'; echo
```

**Expected:**

```text
{"result":{"id":"...","status":"active"},"success":true,"errors":[],"messages":[{"code":10000,"message":"This API Token is valid and active","type":null}]}
```

- `"success":false` - the token is wrong or was not copied in full.
- The command times out or returns an empty response - the Cloudflare API is unreachable from the provider's network. Since June 2025 Russian ISPs have been restricting connections to the Cloudflare network. In that case use the manual option from 6.8.

### 6.4. Installing certbot

```bash
sudo apt install certbot python3-certbot-dns-cloudflare
certbot --version
```

**Expected:** `certbot 1.21.0`. This is the version from the Ubuntu repository. It is older than the current release but supports DNS-01 through the Cloudflare API with a token. Do not install the snap version of certbot next to it: both installations use the `/etc/letsencrypt` directory and run renewals independently of each other.

### 6.5. Test issuance

```bash
sudo certbot certonly --dry-run \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d inkov.dev -d '*.inkov.dev' \
  --email <EMAIL> --agree-tos --no-eff-email
```

Options:

- `--dry-run` - issuance against the Let's Encrypt staging server without saving the certificate. It checks the token, DNS and Let's Encrypt reachability and does not count against the rate limit of 5 identical certificates per week.
- `--dns-cloudflare-propagation-seconds 60` - a pause before validation so the TXT record reaches all Cloudflare DNS servers.
- `-d inkov.dev -d '*.inkov.dev'` - one certificate for the domain and all subdomains. The quotes around `*.inkov.dev` are required, otherwise the shell tries to expand file names.

**Expected:** `The dry run was successful.` It takes about a minute.

### 6.6. Issuing the certificate

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d inkov.dev -d '*.inkov.dev' \
  --email <EMAIL> --agree-tos --no-eff-email \
  --deploy-hook 'prosodyctl --root cert import /etc/letsencrypt/live'
```

`--deploy-hook` runs after every successful issuance and renewal. `prosodyctl --root cert import` copies the certificate and key to `/etc/prosody/certs` with owner `prosody` and reloads Prosody. Do not point Prosody at the files in `/etc/letsencrypt` directly: the key there is readable only by root, and Prosody runs as the `prosody` user.

**Expected:** the output contains the lines

```text
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/inkov.dev/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/inkov.dev/privkey.pem
```

and the import command message `Imported certificate and key for hosts inkov.dev, *.inkov.dev`.

Check:

```bash
sudo certbot certificates
sudo ls -l /etc/prosody/certs/
```

**Expected:** `Domains: inkov.dev *.inkov.dev`, `Expiry Date` about 90 days from now. `/etc/prosody/certs` contains `inkov.dev.crt` and `inkov.dev.key` owned by `prosody`.

### 6.7. Automatic renewal

certbot from the Ubuntu package installs a systemd timer that runs `certbot renew` twice a day. The certificate is renewed 30 days before it expires, and `--deploy-hook` runs after the renewal.

```bash
# Renewal timer
systemctl list-timers certbot.timer --no-pager
# The import command is saved in the renewal settings
sudo grep renew_hook /etc/letsencrypt/renewal/inkov.dev.conf
# Test renewal against the staging server
sudo certbot renew --dry-run
```

**Expected:** `certbot.timer` is listed with the time of the next run; the line `renew_hook = prosodyctl --root cert import /etc/letsencrypt/live`; the message `Congratulations, all simulated renewals succeeded`.

With `--dry-run` the deploy hook does not run. The import command is tested separately in section 9.2.

> **Important:** since June 2025 Let's Encrypt no longer sends certificate expiry emails. If renewal stops working (for example, the Cloudflare API becomes unreachable from the VPS), nobody will tell you. Check the expiry date once a month with `sudo certbot certificates` or set up external monitoring for the certificate at `https://upload.inkov.dev:5281`.

### 6.8. Fallback options

**Option 1. Manual DNS-01 validation.** Use it if the Cloudflare API is unreachable from the VPS. You create the TXT records by hand in the Cloudflare dashboard, and there is no automatic renewal.

```bash
sudo certbot certonly --manual --preferred-challenges dns \
  -d inkov.dev -d '*.inkov.dev' \
  --email <EMAIL> --agree-tos --no-eff-email \
  --deploy-hook 'prosodyctl --root cert import /etc/letsencrypt/live'
```

certbot prints two values for the name `_acme-challenge.inkov.dev`, one at a time, one for each name in the certificate. For each value create a record in Cloudflare: Type `TXT`, Name `_acme-challenge`, Content - the value from the output. Before you press Enter for the last time, check that both records are visible:

```bash
dig +short @"$(dig +short NS inkov.dev | head -1)" TXT _acme-challenge.inkov.dev
```

After issuance, delete the TXT records. `certbot renew` does not work for such a certificate: repeat the command by hand before it expires, roughly every 60 days.

**Option 2. Another certificate authority.** Use it if Let's Encrypt is unreachable but the Cloudflare API works. ZeroSSL supports ACME and wildcard certificates but requires EAB keys from the ZeroSSL account dashboard. Add these options to the command from 6.6:

```bash
  --server https://acme.zerossl.com/v2/DV90 \
  --eab-kid <EAB_KID> --eab-hmac-key <EAB_HMAC_KEY>
```

**Option 3. Self-signed certificate.** Only for testing your own client: federation with other servers will not work, and clients will show a warning.

```bash
sudo prosodyctl cert generate inkov.dev
```

---

## 7. Prosody configuration

### 7.1. Backup

```bash
sudo cp /etc/prosody/prosody.cfg.lua /etc/prosody/prosody.cfg.lua.orig
```

Rollback: `sudo cp /etc/prosody/prosody.cfg.lua.orig /etc/prosody/prosody.cfg.lua && sudo systemctl restart prosody`.

In the Prosody 13 package the whole configuration lives in one file, `/etc/prosody/prosody.cfg.lua`. The `conf.avail` and `conf.d` directories from older Debian packages are not used.

### 7.2. TURN secret

```bash
sudo install -d -m 700 /root/.secrets
sudo sh -c 'umask 077; openssl rand -hex 32 > /root/.secrets/turn_secret'
```

The secret is inserted into the Prosody (7.4) and coturn (8.2) configurations with `sed`, so you do not need to copy it by hand.

### 7.3. What the settings do

| Parameter | Value | Purpose |
|---|---|---|
| `admins` | `admin@inkov.dev` | server administrator: commands from the client, invitations |
| `modules_enabled` | the list from the package, plus `mam` and `turn_external` | server features, explained below |
| `allow_registration` | `false` | registration from clients is closed. Accounts are created by the administrator or through invitations (`invites*` modules) |
| `c2s_require_encryption`, `s2s_require_encryption` | `true` | connections without TLS are refused |
| `s2s_secure_auth` | `true` | other servers must present a valid certificate |
| `limits` | 10 and 30 KB/s | incoming traffic limit for clients and servers, values from the package |
| `pidfile` | `/var/run/prosody/prosody.pid` | `prosodyctl` uses it to reload the server, including after a certificate import |
| `authentication` | `internal_hashed` | passwords are stored as SCRAM hashes |
| storage | default `internal` | files in `/var/lib/prosody`. SQLite makes sense with dozens of users and large archives |
| `archive_expires_after` | `1y` | history is kept on the server for a year and synced between devices. A shorter period means less data on the server. Example values: `1w`, `30d`, `6 months`, `1y`, `never`. The value `1m` is ambiguous (month or minute), and Prosody 13 logs it as an error |
| `http_interfaces` | `127.0.0.1` | plain HTTP on port 5280 is reachable only from the server itself |
| `https_ports` | `5281` | HTTPS for file sharing |
| `turn_external_*` | `turn.inkov.dev`, secret, TCP | clients get the coturn address and temporary credentials valid for 24 hours |
| `certificates` | `certs` | Prosody finds the certificate in `/etc/prosody/certs` by itself, for subdomains through the wildcard |

Modules that mobile clients depend on:

- `smacks` - resumes the session after a network change without losing messages;
- `carbons` - copies messages to all of the user's devices;
- `csi_simple` - holds back non-urgent traffic while the app is in the background;
- `cloud_notify` - push notifications (XEP-0357), shipped with Prosody 13 and enabled by default;
- `mam` - server-side message archive.

Components:

- `conference.inkov.dev` (`muc`) - group chats. `muc_mam` keeps room history, `restrict_room_creation = "local"` lets only `inkov.dev` users create rooms.
- `upload.inkov.dev` (`http_file_share`) - file sharing. This is a separate component, not a module from `modules_enabled`. Limits: files up to 100 MB, up to 1 GB per user per day, up to 5 GB on the server, files kept for 30 days. Large files are streamed to disk, so there is no need to change `http_max_content_size`.
- `proxy.inkov.dev` (`proxy65`) - direct file transfer for old clients. Modern clients use `http_file_share`, so the component is commented out. To enable it, remove `--` from the two lines, create the `proxy` DNS record (section 3.1) and open the port: `sudo ufw allow 5000/tcp comment 'XMPP proxy65'`.

### 7.4. Configuration file

Copy the whole block and run it in the terminal: the command replaces the contents of the file.

```bash
sudo tee /etc/prosody/prosody.cfg.lua > /dev/null <<'EOF'
-- /etc/prosody/prosody.cfg.lua
-- Personal XMPP server for addresses like user@inkov.dev.
-- Based on the configuration from the Prosody 13.0 package.
-- After every edit: sudo prosodyctl check config

---------- Global settings ----------
-- Apply to the whole server. Must come before the first VirtualHost or Component line.

-- Administrators. The account has to be created separately with prosodyctl adduser.
admins = { "admin@inkov.dev" }

modules_enabled = {
    -- Required
        "disco"; -- discovery of server features
        "roster"; -- contact list
        "saslauth"; -- authentication
        "tls"; -- connection encryption

    -- Recommended
        "blocklist"; -- blocking users
        "bookmarks"; -- syncs the list of group chats between clients
        "carbons"; -- copies of messages on all of the user's devices
        "dialback"; -- fallback way to verify other servers via DNS
        "limits"; -- rate limiting for incoming connections
        "pep"; -- account data: avatars, OMEMO keys
        "private"; -- legacy storage for client settings (XEP-0049)
        "smacks"; -- connection recovery without losing messages (XEP-0198)
        "vcard4"; -- user profiles
        "vcard_legacy"; -- compatibility with the old profile format

    -- Mobile clients and convenience
        "account_activity"; -- time of the last login to the account
        "cloud_notify"; -- push notifications for mobile clients (XEP-0357)
        "csi_simple"; -- saves traffic and battery on phones
        "invites"; -- invitations
        "invites_adhoc"; -- creating invitations from the client
        "invites_register"; -- registration by invitation while registration is closed
        "ping"; -- replies to XMPP ping
        "register"; -- password change from the client; allow_registration closes registration
        "time"; -- server time
        "uptime"; -- server uptime
        "version"; -- server version
        "mam"; -- server-side message archive (XEP-0313)
        "turn_external"; -- TURN server details for calls (XEP-0215)

    -- Administration
        "admin_adhoc"; -- administrator commands from the client
        "admin_shell"; -- sudo prosodyctl shell console
}

-- Registration from clients is disabled. Accounts are created by the administrator
-- or through invitations.
allow_registration = false

-- Encryption is required for clients and servers
c2s_require_encryption = true
s2s_require_encryption = true

-- Other servers must present a valid certificate
s2s_secure_auth = true

-- Incoming traffic rate limits (values from the package configuration)
limits = {
    c2s = {
        rate = "10kb/s";
    };
    s2sin = {
        rate = "30kb/s";
    };
}

-- Needed by prosodyctl: it uses this file to find the process to reload,
-- including after a certificate update
pidfile = "/var/run/prosody/prosody.pid"

-- Passwords are stored as hashes (SCRAM)
authentication = "internal_hashed"

-- Default storage: files in /var/lib/prosody. Enough for a personal server.
--storage = "internal"

-- Retention period for the archive of one-to-one messages
archive_expires_after = "1y"

-- HTTP. Plain HTTP (5280) is reachable only from the server itself.
-- HTTPS for file sharing is on port 5281 because 443 is taken by Xray.
http_ports = { 5280 }
http_interfaces = { "127.0.0.1" }
https_ports = { 5281 }

-- TURN server for calls (coturn). The secret matches static-auth-secret in /etc/turnserver.conf.
-- If you do not need calls, delete these three lines and "turn_external" from modules_enabled.
turn_external_host = "turn.inkov.dev"
turn_external_secret = "<TURN_SECRET>"
turn_external_tcp = true

-- Logs
log = {
    info = "/var/log/prosody/prosody.log"; -- change info to debug for a verbose log
    error = "/var/log/prosody/prosody.err";
}

-- Certificate directory relative to this file: /etc/prosody/certs
certificates = "certs"

----------- Virtual host -----------
-- Domain in user addresses: user@inkov.dev
VirtualHost "inkov.dev"

------------- Components -------------

-- Group chats: rooms like room@conference.inkov.dev
Component "conference.inkov.dev" "muc"
    modules_enabled = { "muc_mam" } -- message archive in rooms
    restrict_room_creation = "local" -- only inkov.dev users can create rooms
    muc_log_expires_after = "1y" -- retention period for the room archive

-- File sharing: https://upload.inkov.dev:5281/
Component "upload.inkov.dev" "http_file_share"
    http_file_share_size_limit = 100 * 1024 * 1024 -- maximum file size: 100 MB
    http_file_share_daily_quota = 1024 * 1024 * 1024 -- per user per day: 1 GB
    http_file_share_global_quota = 5 * 1024 * 1024 * 1024 -- total on the server: 5 GB
    http_file_share_expires_after = "30d" -- files are deleted after 30 days

-- Proxy65 (optional): direct file transfer for old clients.
-- To enable it: remove "--" from the two lines below, create the proxy DNS record
-- and open port 5000/tcp.
--Component "proxy.inkov.dev" "proxy65"
--    proxy65_acl = { "inkov.dev" }
EOF
```

Insert the secret, set permissions and check:

```bash
# TURN secret from the file instead of <TURN_SECRET>
sudo sh -c 'sed -i "s/<TURN_SECRET>/$(cat /root/.secrets/turn_secret)/" /etc/prosody/prosody.cfg.lua'
# No placeholders left: 0 expected
sudo grep -c '<TURN_SECRET>' /etc/prosody/prosody.cfg.lua
# Owner root, group prosody, no access for others
sudo chown root:prosody /etc/prosody/prosody.cfg.lua
sudo chmod 640 /etc/prosody/prosody.cfg.lua
# Check syntax and options
sudo prosodyctl check config
```

**Expected output of** `check config`:

```text
Checking config...
    The following configuration files have been loaded:
      -  /etc/prosody/prosody.cfg.lua

    Some of your hosts may be missing features due to a lack of configuration.
    For more details, use the 'prosodyctl check features' command.
Done.

All checks passed, congratulations!
```

The `check features` message is about browser connections (section 9.3) and does not affect the server. The new configuration takes effect on the restart in section 9.1.

---

## 8. Calls: coturn

If you do not need calls, skip this section. In that case remove the `"turn_external";` line and the three `turn_external_*` lines from `/etc/prosody/prosody.cfg.lua`.

### 8.1. Installation

```bash
sudo apt install coturn
# The service starts right after installation with the configuration file from the package.
# Stop it until it is configured.
sudo systemctl stop coturn
```

> **Important:** with the configuration file from the package coturn allocates relay ports without checking credentials. That is why the service is stopped right after installation, and the TURN ports in the firewall are opened only after configuration (section 8.3).

### 8.2. Configuration

```bash
# Backup of the file from the package
sudo cp /etc/turnserver.conf /etc/turnserver.conf.orig
sudo tee /etc/turnserver.conf > /dev/null <<'EOF'
# /etc/turnserver.conf - TURN server for XMPP calls

# STUN/TURN port (UDP and TCP)
listening-port=3478

# UDP port range for media. Must match the firewall rule.
min-port=50000
max-port=50100

# Only if the provider uses NAT (the public IP is not assigned to the VPS interface):
#external-ip=<IP_VPS>/<INTERNAL_IP>

# Temporary credentials are issued by Prosody (mod_turn_external).
# The secret matches turn_external_secret in /etc/prosody/prosody.cfg.lua.
use-auth-secret
static-auth-secret=<TURN_SECRET>
realm=turn.inkov.dev

# TLS is not used (see section 8.4)
no-tls
no-dtls

# Security
fingerprint
no-cli
no-multicast-peers
no-software-attribute

# Deny relaying to local, private and special-purpose addresses.
# Without this, TURN can be used to reach services inside the VPS, including the 3x-ui panel.
denied-peer-ip=0.0.0.0-0.255.255.255
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=100.64.0.0-100.127.255.255
denied-peer-ip=127.0.0.0-127.255.255.255
denied-peer-ip=169.254.0.0-169.254.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
denied-peer-ip=192.0.0.0-192.0.0.255
denied-peer-ip=192.0.2.0-192.0.2.255
denied-peer-ip=192.88.99.0-192.88.99.255
denied-peer-ip=192.168.0.0-192.168.255.255
denied-peer-ip=198.18.0.0-198.19.255.255
denied-peer-ip=198.51.100.0-198.51.100.255
denied-peer-ip=203.0.113.0-203.0.113.255
denied-peer-ip=224.0.0.0-255.255.255.255
denied-peer-ip=::1
denied-peer-ip=::ffff:0.0.0.0-::ffff:255.255.255.255
denied-peer-ip=64:ff9b::-64:ff9b::ffff:ffff
denied-peer-ip=100::-100::ffff:ffff:ffff:ffff
denied-peer-ip=2001::-2001:1ff:ffff:ffff:ffff:ffff:ffff:ffff
denied-peer-ip=2002::-2002:ffff:ffff:ffff:ffff:ffff:ffff:ffff
denied-peer-ip=fc00::-fdff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
denied-peer-ip=fe80::-febf:ffff:ffff:ffff:ffff:ffff:ffff:ffff

# Log to the systemd journal: journalctl -u coturn
syslog
EOF
# TURN secret from the file instead of <TURN_SECRET>
sudo sh -c 'sed -i "s/<TURN_SECRET>/$(cat /root/.secrets/turn_secret)/" /etc/turnserver.conf'
# Only root and the turnserver group can read the file
sudo chown root:turnserver /etc/turnserver.conf
sudo chmod 640 /etc/turnserver.conf
```

If section 2.5 showed that the VPS is behind NAT, uncomment the `external-ip` line and set the public and internal addresses: `sudo nano /etc/turnserver.conf`.

What the options do:

- `use-auth-secret` and `static-auth-secret` - Prosody gives clients a temporary username and password derived from the shared secret. No permanent TURN passwords are needed.
- `min-port` and `max-port` - 101 ports for media. One call through TURN takes several ports, which is enough for a personal server.
- `no-cli` disables the coturn management console, `no-software-attribute` hides the coturn version in responses.
- `denied-peer-ip` - blocks relaying to internal addresses. Without it a TURN user can reach services that listen only on `127.0.0.1` or on the provider's internal network.

### 8.3. Startup and verification

```bash
sudo systemctl start coturn
# Start on boot: enabled expected
systemctl is-enabled coturn
systemctl status coturn --no-pager
sudo ss -tulpn | grep turnserver
```

**Expected:** `Active: active (running)`, `turnserver` listens on port 3478 over UDP and TCP.

TURN ports in the firewall: STUN/TURN over TCP and UDP, plus the media range.

```bash
sudo ufw allow 3478 comment 'STUN/TURN'
sudo ufw allow 50000:50100/udp comment 'TURN relay'
sudo ufw status | grep -E '3478|50000'
```

If the provider has an external firewall (section 2.4), open the same ports there.

Check that Prosody and coturn work together. The command reads the `turn_external_*` settings from the Prosody configuration, so Prosody does not need a restart for it. The name `turn.inkov.dev` must already resolve (section 3).

```bash
# Credential issuance and relay port allocation
sudo prosodyctl check turn -v
# Same, plus relaying a packet to an external STUN server
sudo prosodyctl check turn -v --ping=stun.conversations.im
```

**Expected:**

```text
Identified 1 TURN services.

Testing TURN service turn.inkov.dev:3478...
External IP: 203.0.113.10
Relayed address 1: 203.0.113.10:50073
Success!

All checks passed, congratulations!
```

The port in `Relayed address` is in the 50000-50100 range. The warning `STUN returned a private IP! Is the TURN server behind a NAT and misconfigured?` means the VPS is behind NAT and needs the `external-ip` option.

### 8.4. TLS for coturn (optional)

TURN over TLS on port 5349 helps clients on networks where UDP is blocked but TLS over TCP is allowed. Port 443 cannot be used for this: Xray holds it.

A script that copies the certificate for coturn. certbot runs the scripts in `/etc/letsencrypt/renewal-hooks/deploy` after every renewal.

```bash
sudo install -d -o root -g turnserver -m 750 /etc/coturn/certs
sudo install -d /etc/letsencrypt/renewal-hooks/deploy
sudo tee /etc/letsencrypt/renewal-hooks/deploy/coturn.sh > /dev/null <<'EOF'
#!/bin/sh
# Copies the Let's Encrypt certificate for coturn and restarts the service
set -e
install -o turnserver -g turnserver -m 644 /etc/letsencrypt/live/inkov.dev/fullchain.pem /etc/coturn/certs/fullchain.pem
install -o turnserver -g turnserver -m 600 /etc/letsencrypt/live/inkov.dev/privkey.pem /etc/coturn/certs/privkey.pem
systemctl restart coturn
EOF
sudo chmod 755 /etc/letsencrypt/renewal-hooks/deploy/coturn.sh
```

Enabling TLS in coturn and Prosody:

```bash
# coturn: remove the TLS ban, add the port and certificate
sudo sed -i -e '/^no-tls$/d' -e '/^no-dtls$/d' /etc/turnserver.conf
sudo tee -a /etc/turnserver.conf > /dev/null <<'EOF'

# TLS (section 8.4)
tls-listening-port=5349
cert=/etc/coturn/certs/fullchain.pem
pkey=/etc/coturn/certs/privkey.pem
no-tlsv1
no-tlsv1_1
EOF
# Prosody: tell clients the TLS port of the TURN server
sudo sed -i 's/^turn_external_tcp = true$/&\nturn_external_tls_port = 5349/' /etc/prosody/prosody.cfg.lua
# Firewall: TCP for TLS and UDP for DTLS
sudo ufw allow 5349 comment 'TURN TLS'
# First certificate copy and coturn restart
sudo /etc/letsencrypt/renewal-hooks/deploy/coturn.sh
sudo systemctl reload prosody
```

Check:

```bash
openssl s_client -connect turn.inkov.dev:5349 </dev/null 2>/dev/null | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
```

**Expected:** a line with the expiry date and a line with `DNS:inkov.dev` and `DNS:*.inkov.dev`. On every certificate renewal (about every 60 days) coturn restarts, and calls that go through TURN at that moment are dropped.

---

## 9. Startup and verification

### 9.1. Restarting Prosody

```bash
sudo systemctl restart prosody
systemctl status prosody --no-pager
# Latest log entries and the error file
sudo tail -n 30 /var/log/prosody/prosody.log
sudo cat /var/log/prosody/prosody.err
```

**Expected:** `Active: active (running)`. The log contains these lines:

```text
portmanager	info	Activated service 'c2s' on [*]:5222, [::]:5222
portmanager	info	Activated service 's2s' on [*]:5269, [::]:5269
portmanager	info	Activated service 'http' on [127.0.0.1]:5280
portmanager	info	Activated service 'https' on [*]:5281, [::]:5281
upload.inkov.dev:http	info	Serving 'file_share' at https://upload.inkov.dev:5281/file_share
inkov.dev:tls	info	Certificates loaded
```

`prosody.err` is empty or does not exist. If the service did not start, `journalctl -u prosody -n 50 --no-pager` shows the reason.

### 9.2. Certificate import

certbot runs this same command after every renewal. The configuration now has the `inkov.dev` host and the components, so a manual run works as well:

```bash
sudo prosodyctl --root cert import /etc/letsencrypt/live
```

**Expected:** `Imported certificate and key for hosts inkov.dev, upload.inkov.dev, conference.inkov.dev`. Prosody reloads the certificates by itself, and `Certificates reloaded` appears in the log.

### 9.3. Built-in Prosody checks

```bash
sudo prosodyctl check config
sudo prosodyctl check dns
sudo prosodyctl check certs
sudo prosodyctl check turn -v
sudo prosodyctl check connectivity
sudo prosodyctl check features
```

| Command | What it checks | Normal result |
|---|---|---|
| `check config` | configuration syntax and options | `All checks passed, congratulations!` |
| `check dns` | SRV records, A and AAAA records of the server and components, whether the addresses match the VPS IP | `All checks passed, congratulations!` |
| `check certs` | certificates for `inkov.dev` and the components, names and expiry | `Certificate: /etc/prosody/certs/inkov.dev.crt` for every host |
| `check turn -v` | TURN credential issuance and relay port allocation | `Success!` |
| `check connectivity` | whether ports 5222 and 5269 are reachable from the internet, using the observe.jabber.network service | `xmpp-client: Works`, `xmpp-server: Works` |
| `check features` | the feature set offered to clients | every item `OK` except `Web connections` |

What is specific to this setup:

- `check dns` checks the SRV targets. When SRV records exist, the `inkov.dev` A records with the GitHub Pages addresses are not checked. The message `inkov.dev A record points to unknown address 185.199.x.x` means the server does not see the SRV records: go back to section 3.3.
- `check features` flags `(!) Web connections`: this is BOSH and WebSocket for browser clients. Mobile and desktop clients do not need them.
- `check connectivity` calls an external service. If that service is unreachable from the provider's network, the message `Failed to request check at API` does not mean anything is wrong with the server.
- If the VPS is behind NAT, `check dns` reports that the addresses from DNS are not found on the server. Set the public address in the global part of the configuration, above the `VirtualHost` line: `external_addresses = { "<IP_VPS>" }`.

### 9.4. Checking from outside

Run on: a local computer with Linux or macOS. On Windows, check the ports in PowerShell: `Test-NetConnection xmpp.inkov.dev -Port 5222`, expected `TcpTestSucceeded : True`.

```bash
# SRV records through a regular resolver
dig +short SRV _xmpp-client._tcp.inkov.dev
dig +short SRV _xmpp-server._tcp.inkov.dev

# Port reachability
nc -vz xmpp.inkov.dev 5222
nc -vz xmpp.inkov.dev 5269
nc -vz upload.inkov.dev 5281

# Certificate on the client port: must contain inkov.dev
openssl s_client -connect xmpp.inkov.dev:5222 -starttls xmpp -xmpphost inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
# Certificate on the server port
openssl s_client -connect xmpp.inkov.dev:5269 -starttls xmpp-server -xmpphost inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
# HTTPS certificate for file sharing
openssl s_client -connect upload.inkov.dev:5281 -servername upload.inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
```

**Expected:** SRV records as in section 3.3; `nc` reports a successful connection (`Connected to` or `succeeded`); for each port you get the certificate expiry date and a line with `DNS:inkov.dev` and `DNS:*.inkov.dev`.

### 9.5. External test services

- https://compliance.conversations.im checks support for the XMPP extensions that matter for mobile clients. The service logs in to the server with an account, so use a temporary `test@inkov.dev` (section 10.1) and delete it afterwards.
- Whether external services are reachable from Russia and from the provider's network may change. The main check is the commands from 9.3 and 9.4.

---

## 10. Users and clients

### 10.1. Accounts

```bash
# Administrator (the address listed in admins). The password is typed twice and does not end up in the shell history
sudo prosodyctl adduser admin@inkov.dev
# Test user
sudo prosodyctl adduser test@inkov.dev
# List of users
sudo prosodyctl shell user list inkov.dev
# Effective role of the administrator
sudo prosodyctl shell user role admin@inkov.dev
```

**Expected:** `OK: Created admin@inkov.dev with role 'prosody:member'` - this is the role at creation time. Administrator rights come from the `admins` list, so `user role` shows `OK: prosody:operator`.

Other operations:

```bash
# Change password
sudo prosodyctl passwd test@inkov.dev
# Delete account
sudo prosodyctl deluser test@inkov.dev
```

Invitation (optional). The person sets their own password, and the administrator does not need to know it:

```bash
sudo prosodyctl shell invite create_account friend@inkov.dev
```

**Expected:** `OK: xmpp:friend@inkov.dev?register;preauth=...`. The link opens in Conversations and clients based on it (Monocles Chat, Cheogram): they create an account with a password the user chooses. The invitation lifetime is set with the `--expires-after` flag.

### 10.2. Clients

| Platform | Client | Where to get it |
|---|---|---|
| Android | Conversations | F-Droid: https://f-droid.org/packages/eu.siacs.conversations/, website https://conversations.im |
| Android | Monocles Chat | F-Droid: https://f-droid.org/packages/de.monocles.chat/ |
| Android | Cheogram | https://cheogram.com |
| iOS, macOS | Monal | App Store, website https://monal-im.org |
| Linux | Dino | https://dino.im, distribution packages or Flathub |
| Linux, Windows | Gajim | https://gajim.org |

Conversations is paid on Google Play, and Google Play purchases cannot be paid for from Russia since 2022. The F-Droid version is free.

### 10.3. Connecting a client

- Address: `user@inkov.dev`, password from `prosodyctl adduser`.
- You do not need to enter a server and port: the client finds `xmpp.inkov.dev:5222` through the SRV record.
- If the client cannot find the server (for example, DNS on the network does not answer SRV queries), set it manually in the account's advanced settings: server `xmpp.inkov.dev`, port `5222`.

### 10.4. Testing features

1. Log in as `admin@inkov.dev` on a phone and as `test@inkov.dev` on a computer.
2. Add each other as contacts and exchange messages.
3. Check OMEMO encryption: Conversations has it on by default, in Gajim and Dino you turn it on in the chat window.
4. Send a photo: the file is uploaded to `https://upload.inkov.dev:5281/...`.
5. Create a group chat: its address will look like `room@conference.inkov.dev`.
6. Call between devices on different networks, for example mobile data and Wi-Fi: this tests TURN.
7. Add a contact from another XMPP server and exchange messages: this tests federation.
8. Delete the test user: `sudo prosodyctl deluser test@inkov.dev`.

---

## 11. Security and maintenance

### 11.1. Access

- Registration is closed (`allow_registration = false`), only the administrator creates accounts.
- Use different long passwords for XMPP, 3x-ui and SSH.
- Port 5280 and the `prosodyctl shell` console are available only locally, do not expose them.
- The Cloudflare token can change DNS in the `inkov.dev` zone. If you suspect a leak, delete it in **My Profile → API Tokens**, create a new one and update `/root/.secrets/cloudflare.ini`.

### 11.2. Updates

- Ubuntu security updates are installed by the `unattended-upgrades` service, which is enabled by default: `systemctl status unattended-upgrades --no-pager`.
- By default `unattended-upgrades` does not update packages from the Prosody repository. Once a month run:

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
```

- The `prosody` package follows the current stable release, including the move to the next major version. Before such an upgrade read the release notes on https://prosody.im. To update Prosody only by hand: `sudo apt-mark hold prosody`, to release the hold: `sudo apt-mark unhold prosody`.

### 11.3. Backups

```bash
# Configuration, Prosody data, certificates and secrets. Files uploaded by users are not included
sudo tar --exclude='*/http_file_share' -czf /root/xmpp-backup-$(date +%F).tar.gz \
  /etc/prosody /var/lib/prosody /etc/letsencrypt /etc/turnserver.conf /root/.secrets
sudo ls -lh /root/xmpp-backup-*.tar.gz
```

The message `tar: Removing leading '/' from member names` is expected. Copying the archive to your local computer:

```bash
# On the VPS: copy the archive to the user's home directory, readable only by that user
sudo install -m 600 -o "$USER" /root/xmpp-backup-$(date +%F).tar.gz ~/
# On the local computer
scp <user>@xmpp.inkov.dev:~/xmpp-backup-*.tar.gz .
# On the VPS: remove the copy from the home directory
rm ~/xmpp-backup-*.tar.gz
```

Restore: `sudo tar -xzf xmpp-backup-<date>.tar.gz -C /` and `sudo systemctl restart prosody coturn`.

> **Important:** the archive contains the certificate's private key, the Cloudflare token and the TURN secret. Store the copy encrypted.

### 11.4. Logs

- Prosody: `/var/log/prosody/prosody.log` and `prosody.err`. The package sets up rotation: daily, 14 files.
- coturn: `journalctl -u coturn`.
- certbot: `/var/log/letsencrypt/letsencrypt.log`.

### 11.5. fail2ban

Optional. By default Prosody 13 does not log the IP addresses of failed login attempts, so fail2ban needs the third-party module `mod_log_auth` (https://modules.prosody.im/mod_log_auth). With long random passwords brute force does not work, so a personal server can do without fail2ban.

### 11.6. Upgrading to Ubuntu 24.04

Standard support for Ubuntu 22.04 ends in 2027. When you upgrade:

1. Make a backup (11.3).
2. `do-release-upgrade` disables third-party repositories. After the upgrade, add the Prosody repository for 24.04:

```bash
sudo wget https://prosody.im/downloads/repos/noble/prosody.sources -O /etc/apt/sources.list.d/prosody.sources
sudo apt update && sudo apt upgrade
```

3. Check the services: `sudo prosodyctl check`, `systemctl status coturn --no-pager`, `sudo certbot renew --dry-run`.

---

## 12. Troubleshooting

| Symptom | Likely cause | How to check | How to fix |
|---|---|---|---|
| Prosody does not start, the log says `Prosody is no longer compatible with Lua 5.1` | `luarocks` or `lua5.1` is installed, the `lua` command points to Lua 5.1 | `readlink -f /usr/bin/lua` | `sudo update-alternatives --set lua-interpreter /usr/bin/lua5.4`, then `sudo systemctl restart prosody` |
| Prosody does not start after a configuration edit | Lua syntax error: a missing quote, bracket or `;` | `sudo prosodyctl check config` shows the line with the error | Fix the line or restore `/etc/prosody/prosody.cfg.lua.orig` |
| `Address already in use` in the log | The port is taken by another process | `sudo ss -tulpn`, find the port in the output | Free the port or change it in the configuration and the SRV records |
| The client cannot find the server | SRV records are missing or not yet visible to the resolver; port 5222 is closed | `dig +short SRV _xmpp-client._tcp.inkov.dev`, `nc -vz xmpp.inkov.dev 5222` | Check the records (section 3), the VPS firewall and the provider's firewall |
| The client reports a certificate error | The certificate does not include `inkov.dev`, was not imported or has expired | The `openssl` commands from 9.4, `sudo prosodyctl check certs` | Issue the certificate with `-d inkov.dev -d '*.inkov.dev'`, run `sudo prosodyctl --root cert import /etc/letsencrypt/live` |
| certbot: `Error determining zone_id: 9109 ... Did you enter a valid Cloudflare Token?` | Wrong token | Token check from 6.3 | Copy the token again or create a new one |
| certbot: `Unable to determine zone_id for inkov.dev` | The token has no access to the zone | Token settings in Cloudflare | Zone Resources: `Include` / `Specific zone` / `inkov.dev` |
| certbot: `Error communicating with the Cloudflare API` with a hint about `Zone:DNS:Edit` | The token is not allowed to edit DNS | Token Permissions | `Zone` / `DNS` / `Edit` |
| certbot or the token check times out | The Cloudflare API is unreachable from the provider's network | Token check from 6.3 | Manual DNS-01 validation (6.8, option 1) |
| certbot: `DNS problem: NXDOMAIN looking up TXT` or `Incorrect TXT record` | The TXT record had not propagated yet | Run the command again | Raise `--dns-cloudflare-propagation-seconds` to 120 |
| Messages to other servers are not delivered | Port 5269 is closed, no `_xmpp-server` SRV record, the remote server has an invalid certificate | `nc -vz xmpp.inkov.dev 5269`, `sudo tail -n 50 /var/log/prosody/prosody.log` | Open the port, check SRV; for a single domain with an invalid certificate add `s2s_insecure_domains = { "example.org" }` to the global part of the configuration |
| Files are not sent | Port 5281 is closed, no `upload` record, a limit is exceeded | `nc -vz upload.inkov.dev 5281`, certificate check on 5281 from 9.4 | Open the port, create the record, change the limits in the `upload.inkov.dev` component |
| Calls do not connect or there is no audio | coturn is stopped, 50000-50100/udp is closed, the secrets do not match, the VPS is behind NAT without `external-ip` | `systemctl status coturn`, `sudo prosodyctl check turn -v --ping=stun.conversations.im` | Start coturn, open the ports, insert the secret again, set `external-ip` |
| `check turn`: `STUN returned a private IP` | The VPS is behind NAT | `ip -br addr` | `external-ip=<IP_VPS>/<INTERNAL_IP>` in `/etc/turnserver.conf`, then `sudo systemctl restart coturn` |
| Prosody: `Permission denied` when loading the key | The configuration points to a key in `/etc/letsencrypt` | `sudo cat /var/log/prosody/prosody.err` | Remove explicit certificate paths, run `cert import` |
| SSH access is lost or the VPN stopped working after enabling ufw | The SSH or 3x-ui ports are not allowed | Provider's web console: `sudo ufw status numbered` | `sudo ufw allow <PORT>/tcp` or `sudo ufw disable` |
| Connections work on some networks but not on others | Non-standard ports are blocked on the client's network | Test from another network | Connections and file sharing over port 443 (Appendix A) |
| `check dns`: `inkov.dev A record points to unknown address 185.199.x.x` | The server's resolver does not see the SRV records | `dig +short SRV _xmpp-client._tcp.inkov.dev` on the VPS | Check the SRV records in Cloudflare, wait up to 30 minutes |

Verbose Prosody log: in `/etc/prosody/prosody.cfg.lua` change `info =` to `debug =` in the `log` block, run `sudo systemctl restart prosody`, reproduce the problem and switch back to `info`.

---

## 13. Summary

1. Prosody 13 on the VPS handles `user@inkov.dev` addresses, the `inkov.dev` site stays on GitHub Pages.
2. SRV records in the `inkov.dev` zone point clients and other servers to `xmpp.inkov.dev`.
3. The certificate for `inkov.dev` and `*.inkov.dev` is issued through DNS-01 and the Cloudflare API, renewed automatically and imported into Prosody.
4. File sharing runs on port 5281, calls go through coturn. Port 443 stays with Xray, the 3x-ui configuration was not changed.
5. Registration is closed, encryption is required, coturn does not relay traffic to internal addresses.
6. Optional: client connections and file sharing over port 443 next to REALITY (Appendix A).

### Checklist

- [ ] Prosody and coturn ports are free, the 3x-ui database is backed up (section 2)
- [ ] DNS records are created as DNS only and visible with `dig` (section 3)
- [ ] Prosody 13 is installed with `--no-install-recommends`, `prosodyctl about` shows Lua 5.4 (section 4)
- [ ] ufw rules are added, SSH and the VPN work (section 5)
- [ ] The certificate is issued, `sudo certbot renew --dry-run` passes (section 6)
- [ ] `sudo prosodyctl check config` reports no errors (section 7)
- [ ] `sudo prosodyctl check turn -v` shows `Success!` (section 8)
- [ ] `check dns`, `check certs`, `check connectivity` report no errors (section 9)
- [ ] Messages, files, group chats, calls and federation work (section 10)
- [ ] A backup is made and copied off the server (section 11)

### Security measures

The server is reachable from the internet, so its security depends on regular maintenance. Once a month update the packages and check the certificate expiry date (`sudo certbot certificates`). Do not expose port 5280 and do not turn off `s2s_secure_auth` or `c2s_require_encryption`. Keep the files from `/root/.secrets` and the backups in a safe place. After any firewall change check SSH access and the VPN.

### Documentation

- Prosody: https://prosody.im/doc
- Prosody package repository: https://prosody.im/download/package_repository
- Configuration: https://prosody.im/doc/configure
- Certificates: https://prosody.im/doc/certificates, Let's Encrypt: https://prosody.im/doc/letsencrypt
- DNS: https://prosody.im/doc/dns
- TURN: https://prosody.im/doc/turn
- mod_http_file_share: https://prosody.im/doc/modules/mod_http_file_share
- mod_turn_external: https://prosody.im/doc/modules/mod_turn_external
- mod_mam: https://prosody.im/doc/modules/mod_mam
- mod_muc: https://prosody.im/doc/modules/mod_muc
- prosodyctl: https://prosody.im/doc/prosodyctl
- Dependencies and Lua versions: https://prosody.im/doc/depends
- coturn: https://github.com/coturn/coturn
- certbot: https://eff-certbot.readthedocs.io/en/stable/
- certbot plugin for Cloudflare: https://certbot-dns-cloudflare.readthedocs.io/en/stable/
- Cloudflare DNS: https://developers.cloudflare.com/dns/
- Cloudflare API tokens: https://developers.cloudflare.com/fundamentals/api/get-started/create-token/
- REALITY in Xray: https://xtls.github.io/en/config/transports/reality.html
- HAProxy 2.4: https://docs.haproxy.org/2.4/configuration.html

---

## Appendix A. XMPP and file sharing over port 443

An optional addition for clients that connect from networks where only port 443 is allowed. The appendix changes one 3x-ui setting, so do it only after the main setup from sections 1-10 already works.

### A.1. How it works

Xray with REALITY on port 443 forwards every connection that fails the REALITY check to the address in the Target field. Target usually points to an external site used as cover. In this setup Target is changed to a local HAProxy. HAProxy reads the server name (SNI) from the plaintext part of the TLS handshake, does not decrypt the traffic and routes the connection by name:

```text
Client ──443──> Xray (REALITY)
                ├── VPN client (REALITY check passed) ──> VPN, as before
                └── other connections ──> Target: HAProxy 127.0.0.1:8443
                                           ├── SNI inkov.dev        ──> Prosody 127.0.0.1:5223, Direct TLS
                                           ├── SNI upload.inkov.dev ──> Prosody 127.0.0.1:5281, HTTPS
                                           └── any other SNI        ──> original Target (cover site)
```

TLS is terminated by Prosody itself with its own Let's Encrypt certificate; Xray and HAProxy pass the bytes through unchanged. The following starts working over port 443:

- client connections over Direct TLS (XEP-0368): a new SRV record `_xmpps-client._tcp` points to port 443;
- file sharing: file links look like `https://upload.inkov.dev/file_share/...`, without a port number.

Server-to-server connections (5269) and calls (3478 and 50000-50100/udp) stay on their current ports.

### A.2. Requirements and risks

- **This works only for an inbound with Security = reality.** If the inbound on port 443 in 3x-ui has Security = tls, Xray decrypts the connections itself and hands already decrypted traffic to the fallback. That does not work for XMPP clients: Prosody receives a connection without TLS and, with `c2s_require_encryption = true`, refuses the login.
- **The VPN starts depending on HAProxy.** REALITY contacts Target on every connection, VPN client connections included. On the test bench, VPN clients could not connect while HAProxy was stopped.
- **The 3x-ui configuration changes:** one field, Target. This is an exception to the principle from section 1.3.
- Prosody sees clients connected through 443 as coming from 127.0.0.1. The real IP addresses of these clients do not appear in the Prosody log.
- The traffic does not become hidden: the names `inkov.dev`, `upload.inkov.dev` and the XMPP protocol marker (ALPN `xmpp-client`) are sent in the plaintext part of the TLS handshake. Only the port changes.
- If the REALITY settings limit the speed of connections that fail the check (`limitFallbackUpload`, `limitFallbackDownload`, off by default), these limits also apply to XMPP over 443.

The setup was tested on a test bench: Xray 26.3.27 with REALITY, HAProxy 2.4 from Ubuntu 22.04, Prosody 13.0.7. The following worked over port 443: XMPP client login, upload slot request and file upload, VPN client connection, the cover site answering a foreign SNI and a connection without SNI, a ClientHello of about 1.5 KB and a ClientHello that arrived in two parts.

### A.3. Current REALITY settings

Run on: the 3x-ui panel.

1. Open **Inbounds**, find the inbound on port 443 and open it for editing.
2. Check that Security = `reality`.
3. Write down the values of the **Target** field (**Dest** in older panel versions) and the **SNI** field. Target is an address with a port, for example `www.microsoft.com:443`. Below, this value is called `<REALITY_TARGET>`, and the first name from SNI is called `<REALITY_SNI>`.
4. Check that **Xver** = `0`. With any other value Xray adds a PROXY header to the connection, and HAProxy with this configuration will not be able to read the SNI.

Do not change anything at this step.

### A.4. HAProxy

```bash
sudo apt install haproxy
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<'EOF'
# /etc/haproxy/haproxy.cfg - SNI routing behind REALITY (Appendix A)

global
    log /dev/log local0
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    # For established connections: XMPP sessions last for hours
    timeout tunnel 1h

# REALITY sends the connections that failed its check here
frontend reality_target
    bind 127.0.0.1:8443
    # Wait for the TLS ClientHello to read the SNI. Traffic is not decrypted
    tcp-request inspect-delay 5s
    tcp-request content accept if { req.ssl_hello_type 1 }
    use_backend prosody_c2s if { req.ssl_sni -i inkov.dev }
    use_backend prosody_upload if { req.ssl_sni -i upload.inkov.dev }
    default_backend reality_original_target

# XMPP clients with Direct TLS (XEP-0368)
backend prosody_c2s
    server prosody 127.0.0.1:5223

# File sharing
backend prosody_upload
    server prosody 127.0.0.1:5281

# Original Target from the REALITY settings: the cover site
backend reality_original_target
    server target <REALITY_TARGET>
EOF
# Original Target value from 3x-ui instead of the placeholder. Replace the example value with yours
sudo sed -i 's|<REALITY_TARGET>|www.microsoft.com:443|' /etc/haproxy/haproxy.cfg
# Check syntax and start
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
systemctl is-enabled haproxy
```

**Expected:** `Configuration file is valid` and `enabled`.

If the original Target points to local port 8443 (for example, `127.0.0.1:8443` or a bare `8443`), pick another free port for HAProxy and use it in the `bind` line and in A.6.

Check the route to the original Target. At this step the VPN still works the old way:

```bash
openssl s_client -connect 127.0.0.1:8443 -servername <REALITY_SNI> </dev/null 2>/dev/null | openssl x509 -noout -subject
```

**Expected:** the `subject` of the certificate of the site from Target. Empty output means HAProxy could not connect to Target: check the address in `/etc/haproxy/haproxy.cfg` and the log `sudo tail -n 20 /var/log/haproxy.log`.

### A.5. Prosody

```bash
# Backup before the changes
sudo cp /etc/prosody/prosody.cfg.lua /etc/prosody/prosody.cfg.lua.before-443
# Global part: the lines are added after https_ports
sudo sed -i '/^https_ports = { 5281 }$/r /dev/stdin' /etc/prosody/prosody.cfg.lua <<'EOF'

-- Direct TLS for clients (XEP-0368). Connections arrive through port 443
-- from HAProxy (Appendix A), so the port listens on 127.0.0.1 only
c2s_direct_tls_ports = { 5223 }
c2s_direct_tls_interfaces = { "127.0.0.1" }
-- HAProxy passes TLS through unchanged and does not add X-Forwarded-For,
-- so the header coming from 127.0.0.1 must not be trusted
trusted_proxies = { }
EOF
# File sharing component: file links without port 5281
sudo sed -i '/^Component "upload.inkov.dev" "http_file_share"$/r /dev/stdin' /etc/prosody/prosody.cfg.lua <<'EOF'
    http_external_url = "https://upload.inkov.dev/" -- file links through port 443
EOF
# Check and apply
sudo grep -n -E 'c2s_direct_tls|trusted_proxies|http_external_url' /etc/prosody/prosody.cfg.lua
sudo prosodyctl check config
sudo systemctl restart prosody
sudo grep -E "c2s_direct_tls' on|Serving 'file_share'" /var/log/prosody/prosody.log | tail -n 2
```

**Expected:** `grep` finds four lines: `c2s_direct_tls_ports`, `c2s_direct_tls_interfaces`, `trusted_proxies`, `http_external_url`. `check config` prints `All checks passed, congratulations!`. The log has two lines, their order may differ:

```text
upload.inkov.dev:http	info	Serving 'file_share' at https://upload.inkov.dev/file_share
portmanager	info	Activated service 'c2s_direct_tls' on [127.0.0.1]:5223
```

Check the HAProxy routes to Prosody. At this step the VPN still works the old way:

```bash
# File sharing: Prosody certificate
openssl s_client -connect 127.0.0.1:8443 -servername upload.inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep DNS:
# Direct TLS: Prosody answers with the list of login mechanisms
(printf "<?xml version='1.0'?><stream:stream to='inkov.dev' xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' version='1.0'>"; sleep 2) \
  | timeout 5 openssl s_client -connect 127.0.0.1:8443 -servername inkov.dev -alpn xmpp-client -quiet 2>/dev/null \
  | grep -o '<mechanism>[A-Z0-9-]*</mechanism>'
```

**Expected:** a line with `DNS:inkov.dev` and `DNS:*.inkov.dev`, then lines like `<mechanism>SCRAM-SHA-1</mechanism>`.

### A.6. Switching Target in 3x-ui

> **Important:** after this step the VPN goes through HAProxy. Have a client device ready to test the VPN, and keep the original Target value you wrote down at hand.

1. In 3x-ui open the same inbound on port 443.
2. Set the **Target** (**Dest**) field to `127.0.0.1:8443`. Do not change SNI, keys or Short IDs, leave Xver at `0`.
3. Save the inbound. If the panel did not apply the changes by itself, restart Xray from the panel.
4. Test the VPN connection from the client device right away.

If the VPN does not connect, put the original value back into Target and check HAProxy: `systemctl status haproxy --no-pager` and the check from A.4.

### A.7. DNS

Run on: the Cloudflare dashboard, **DNS → Records**.

| Action | Type | Name | Priority | Weight | Port | Target |
|---|---|---|---|---|---|---|
| Add | SRV | `_xmpps-client._tcp` | `0` | `5` | `443` | `xmpp.inkov.dev` |
| Edit | SRV | `_xmpp-client._tcp` | `10` instead of `0` | `5` | `5222` | `xmpp.inkov.dev` |

Clients that support XEP-0368 merge the `_xmpps-client` and `_xmpp-client` records and connect in priority order: port 443 first, then 5222 if that fails. Clients without XEP-0368 support keep connecting to 5222.

```bash
NS=$(dig +short NS inkov.dev | head -1)
dig +short @"$NS" SRV _xmpps-client._tcp.inkov.dev
dig +short @"$NS" SRV _xmpp-client._tcp.inkov.dev
```

**Expected:** `0 5 443 xmpp.inkov.dev.` and `10 5 5222 xmpp.inkov.dev.`

### A.8. Check

Run on: your local computer.

```bash
# XMPP over port 443
(printf "<?xml version='1.0'?><stream:stream to='inkov.dev' xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' version='1.0'>"; sleep 3) \
  | timeout 6 openssl s_client -connect xmpp.inkov.dev:443 -servername inkov.dev -alpn xmpp-client -quiet 2>/dev/null \
  | grep -o '<mechanism>[A-Z0-9-]*</mechanism>'
# File sharing over port 443
openssl s_client -connect xmpp.inkov.dev:443 -servername upload.inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
# A foreign SNI still gets the cover site
openssl s_client -connect xmpp.inkov.dev:443 -servername <REALITY_SNI> </dev/null 2>/dev/null \
  | openssl x509 -noout -subject
```

**Expected:** lines with login mechanisms; the expiry date and `DNS:inkov.dev`, `DNS:*.inkov.dev`; for `<REALITY_SNI>`, the certificate of the site from the original Target, same as before the changes.

On the VPS:

```bash
sudo prosodyctl check dns
sudo prosodyctl check connectivity
```

- `check dns` reports `SRV target xmpp.inkov.dev contains unknown Direct TLS client port: 443`. This is expected: Prosody listens on port 5223, and port 443 is served by Xray and HAProxy.
- `check connectivity` also tests Direct TLS and prints `xmpps-client: Works`.

In the client, reconnect the account and send a file: the link starts with `https://upload.inkov.dev/file_share/`, without `:5281`. Links sent earlier with port 5281 keep working because port 5281 stays open.

### A.9. Maintenance

- HAProxy resolves the Target IP address at startup and when the configuration is reloaded. If the cover site changes its address, the VPN stops connecting. A weekly configuration reload, which does not drop established connections:

```bash
echo '0 5 * * 1 root systemctl reload haproxy' | sudo tee /etc/cron.d/haproxy-reload
```

- The cover site is now set in `backend reality_original_target` in `/etc/haproxy/haproxy.cfg`. To switch to another site, change this address and the SNI in 3x-ui, and leave Target in 3x-ui at `127.0.0.1:8443`.
- HAProxy log: `/var/log/haproxy.log`.

### A.10. Rollback

1. 3x-ui: put the original value back into Target.
2. Cloudflare: delete the `_xmpps-client._tcp` record and set the priority of the `_xmpp-client._tcp` record back to `0`.
3. Prosody: `sudo cp /etc/prosody/prosody.cfg.lua.before-443 /etc/prosody/prosody.cfg.lua && sudo systemctl restart prosody`. The copy was made at the start of A.5: any configuration changes made after it have to be repeated.
4. HAProxy: `sudo systemctl disable --now haproxy` and `sudo rm /etc/cron.d/haproxy-reload`.

### A.11. Troubleshooting

| Symptom | Likely cause | How to check | How to fix |
|---|---|---|---|
| The VPN stopped connecting after the Target change | HAProxy is stopped or cannot connect to the original Target | `systemctl status haproxy --no-pager`, the check from A.4 | Start HAProxy, fix the address in `backend reality_original_target` or restore Target (A.10) |
| The VPN stopped connecting after a few days or weeks | The cover site changed its IP address | The check from A.4 | `sudo systemctl reload haproxy`, check the job from A.9 |
| XMPP does not connect over 443 but works over 5222 | No `_xmpps-client` record, Prosody does not listen on 5223, or the client does not support XEP-0368 | The checks from A.5 and A.8 | Fix the record or the configuration. Clients without XEP-0368 connect to 5222 |
| `upload.inkov.dev:443` answers with the cover site | No route in HAProxy or Xver is not 0 | The check from A.5 | Check the `use_backend prosody_upload` line and the Xver field |
| File links still contain `:5281` | `http_external_url` was not added or Prosody was not restarted | The `Serving 'file_share' at` line in the Prosody log | Repeat A.5 |
