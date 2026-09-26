# Step-by-step guide: Prosody XMPP server on a clean VPS in Russia (Ubuntu 24.04)

Versions: Ubuntu 24.04.5 LTS, Prosody 13.0.7 from the packages.prosody.im repository, SQLite driver lua-dbi-sqlite3 0.7.2, coturn 4.6.1 and certbot 2.9.0 with the dns-cloudflare plugin from the Ubuntu repository. DNS for the domain is hosted on Cloudflare. The Prosody and coturn configurations in this guide were tested on these versions.

This guide is for a dedicated server with no other services on it. Differences from the version for a VPS with 3x-ui ([GUIDE_XMPP_Prosody_Ubuntu_22.04_HAProxy_3x-ui.en.md](GUIDE_XMPP_Prosody_Ubuntu_22.04_HAProxy_3x-ui.en.md)):

- Prosody holds port 443 itself: client connections and file sharing both run on it.
- Prosody data is stored in an SQLite database.
- Basic server preparation is included: a sudo user, SSH key-only login, a firewall.
- No steps for 3x-ui and Xray.

> **Important:** this guide is written specifically for Ubuntu 24.04. On Ubuntu 22.04 the `lua-dbi-sqlite3` driver is built only for Lua 5.1-5.3, while Prosody 13 runs on Lua 5.4. SQLite storage does not start there: Prosody runs without storage, accounts cannot be created and login does not work.

## Contents

1. [Introduction](#1-introduction)
2. [Preparing the server](#2-preparing-the-server)
3. [DNS records in Cloudflare](#3-dns-records-in-cloudflare)
4. [Installing Prosody](#4-installing-prosody)
5. [Firewall](#5-firewall)
6. [SSL certificate](#6-ssl-certificate)
7. [Prosody configuration](#7-prosody-configuration)
8. [Calls: coturn](#8-calls-coturn)
9. [Startup and verification](#9-startup-and-verification)
10. [Users and clients](#10-users-and-clients)
11. [Security and maintenance](#11-security-and-maintenance)
12. [Troubleshooting](#12-troubleshooting)
13. [Summary](#13-summary)

---

## 1. Introduction

### 1.1. What XMPP and Prosody are

XMPP is an open messaging protocol. XMPP servers are connected to each other the same way mail servers are: a user on `inkov.dev` can chat with users on any other XMPP server. Prosody is an XMPP server written in Lua. It uses tens of megabytes of memory, is configured with a single file and suits a personal server. Once you complete this guide, the server handles addresses like `user@inkov.dev`, keeps message history, transfers files, and supports group chats and calls.

### 1.2. Layout

```text
Cloudflare DNS, zone inkov.dev
├── inkov.dev                     A      GitHub Pages (unchanged)
├── xmpp.inkov.dev                A      <IP_VPS>
├── conference.inkov.dev          CNAME  xmpp.inkov.dev
├── upload.inkov.dev              CNAME  xmpp.inkov.dev
├── turn.inkov.dev                CNAME  xmpp.inkov.dev
├── _xmpps-client._tcp.inkov.dev  SRV    0 5 443 xmpp.inkov.dev
├── _xmpp-client._tcp.inkov.dev   SRV    10 5 5222 xmpp.inkov.dev
└── _xmpp-server._tcp.inkov.dev   SRV    0 5 5269 xmpp.inkov.dev

VPS <IP_VPS>, Ubuntu 24.04
├── SSH            22/tcp
├── Prosody        443/tcp           clients (Direct TLS) and HTTPS for file sharing
│                  5222/tcp          clients (STARTTLS), fallback port
│                  5269/tcp          server-to-server connections
│                  5280/tcp          127.0.0.1 only
└── coturn         3478/tcp and udp  STUN/TURN for calls
                   50000-50100/udp   media relay
```

How connections work:

1. A client with the address `user@inkov.dev` looks up the SRV records `_xmpps-client._tcp.inkov.dev` and `_xmpp-client._tcp.inkov.dev`. Clients that support XEP-0368 first connect to `xmpp.inkov.dev:443` with TLS from the start, and on failure fall back to port 5222 with STARTTLS. Clients without XEP-0368 support connect to 5222.
2. Prosody presents a certificate for `inkov.dev`.
3. Other XMPP servers find the server through the SRV record `_xmpp-server._tcp.inkov.dev`, port 5269.
4. Files are uploaded and downloaded at URLs like `https://upload.inkov.dev/file_share/...` over port 443.
5. The GitHub Pages site keeps working: SRV records have different names and do not overlap with the `inkov.dev` A records.

### 1.3. Design decisions

- **Addresses `user@inkov.dev`, server on `xmpp.inkov.dev`.** Prosody has `VirtualHost "inkov.dev"`, and SRV records point the domain to the server.
- **Certificate for `inkov.dev` and `*.inkov.dev`.** Clients and other servers check the certificate against the domain in the address (RFC 6120), not against the name from the SRV record. The apex domain `inkov.dev` points to GitHub, so the HTTP challenge (HTTP-01) cannot be passed from the VPS. The DNS challenge (DNS-01) is used instead: certbot creates a temporary TXT record through the Cloudflare API. One wildcard certificate covers `inkov.dev` and all subdomains.
- **Prosody 13 from the official repository.** The Ubuntu 24.04 repository has version 0.12.4 from the previous branch. The official repository provides the current 13.x branch and its fixes.
- **Port 443 belongs to Prosody.** The `mod_net_multiplex` module accepts Direct TLS for clients (XEP-0368) and HTTPS for file sharing on the same port. The service is selected by ALPN, and if the client sent no ALPN, by the first bytes of the connection. Port 5222 stays for clients without XEP-0368 support.
- **SQLite storage.** Accounts, contacts, the message archive and group chat data are stored in one file, `/var/lib/prosody/prosody.sqlite`. A copy of the database is taken on a running server with `sqlite3 .backup`, and moving to another server comes down to copying one file. The `lua-dbi-sqlite3` driver is required (section 4.2). Uploaded files are stored on disk, not in the database.
- **Server-to-server connections over 5269 only.** Certificate verification for other servers is not configured on port 443, and with `s2s_secure_auth = true` servers fail authentication without it. So the `_xmpps-server` record is not published.
- **Calls through coturn.** `mod_turn_external` only hands clients the address and temporary credentials of the TURN server. The TURN server itself is installed separately (section 8).
- **Dedicated server.** Basic server hardening is part of this guide (section 2).

### 1.4. Limitations

- Port 443 is taken by Prosody. To host a website or a VPN on port 443 of this IP later, you will need an SNI router in front of the services (HAProxy, for example).
- Calls use port 3478 and the 50000-50100/udp range. On networks where only port 443 is open, calls may not work.
- Traffic is not hidden: the names `inkov.dev`, `upload.inkov.dev` and the XMPP protocol marker (ALPN `xmpp-client`) are sent in the unencrypted part of the TLS handshake.
- The domain in user addresses can only be changed together with the accounts: after a domain change they have to be created again.

### 1.5. Parameters

| Parameter | Value | Used in |
|---|---|---|
| XMPP domain | `inkov.dev` | user addresses, `VirtualHost` |
| Server host | `xmpp.inkov.dev` | A record, SRV target |
| `<IP_VPS>` | public IPv4 address of the VPS, `203.0.113.10` in the examples | DNS, SSH, checks |
| `<USER>` | name of the sudo user | SSH, section 2.3 |
| `<EMAIL>` | email for the Let's Encrypt account | certbot |
| `<CF_API_TOKEN>` | Cloudflare API token | only in `/root/.secrets/cloudflare.ini` |
| `<TURN_SECRET>` | shared secret for Prosody and coturn | created in section 7.2, inserted by a command |

Commands in code blocks run on the VPS unless stated otherwise. Replace the placeholders in angle brackets with your own values.

### 1.6. What you need

- A VPS with a clean Ubuntu 24.04 LTS and SSH access: root or a sudo user created by the hosting provider.
- An SSH key on your local computer. If you do not have one, it is created in section 2.3.
- Access to the Cloudflare dashboard for the `inkov.dev` domain.
- Access to the VPS console in the provider's control panel (VNC or similar) in case of a mistake in the SSH or firewall setup.
- An XMPP client on a phone or computer (section 10).

---

## 2. Preparing the server

### 2.1. OS and resources

```bash
# OS version: expect Ubuntu 24.04.x LTS
lsb_release -ds
# Free space on the root partition
df -h /
# Free memory
free -h
```

**Expected:** `Ubuntu 24.04.5 LTS` (SQLite storage does not work on Ubuntu 22.04, see the warning at the top), at least 6 GB of free disk space: up to 5 GB is reserved for user files (section 7). If you have less, lower `http_file_share_global_quota` in the Prosody configuration. Prosody and coturn together use less than 100 MB of memory.

### 2.2. APT mirrors and system update

```bash
sudo apt update
```

**Expected:** `Hit:` and `Get:` lines, no `Err:` and no hangs. In that case there is no need to change mirrors: Russian hosting providers often have local ones configured already.

If `apt update` fails with `Err:` lines or hangs, switch the system to the Yandex mirror. In Ubuntu 24.04 the repository list is stored in `/etc/apt/sources.list.d/ubuntu.sources` in deb822 format, and `/etc/apt/sources.list` contains only a comment.

```bash
# Backup of the current list
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak
# New list: main repository, updates, backports and security updates
sudo tee /etc/apt/sources.list.d/ubuntu.sources > /dev/null <<'EOF'
Types: deb
URIs: https://mirror.yandex.ru/ubuntu/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: https://mirror.yandex.ru/ubuntu/
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF
sudo apt update
```

Rollback:

```bash
sudo cp /etc/apt/sources.list.d/ubuntu.sources.bak /etc/apt/sources.list.d/ubuntu.sources && sudo apt update
```

If there is no `ubuntu.sources` file and the `deb` lines are in `/etc/apt/sources.list`, the provider's image uses the old format. In that case rename the old file before creating `ubuntu.sources`, so the repositories are not listed twice: `sudo mv /etc/apt/sources.list /etc/apt/sources.list.bak`.

Package upgrade:

```bash
sudo apt upgrade -y
# Check whether a kernel update requires a reboot
ls /var/run/reboot-required 2>/dev/null && echo "reboot required"
```

If a reboot is required, run `sudo reboot` and reconnect.

### 2.3. Sudo user and key-based login

There is no need to work as root all the time: a separate user with sudo limits the consequences of mistakes. If the provider gave you root access only, create a user:

```bash
# On the VPS as root
adduser <USER>
usermod -aG sudo <USER>
```

`adduser` asks for a password: it is needed for `sudo`. The other fields can be left empty.

Run on: a local computer with Linux or macOS.

```bash
# Create a key if you do not have one yet (file ~/.ssh/id_ed25519)
ssh-keygen -t ed25519
# Copy the public key to the VPS while password login is still allowed
ssh-copy-id <USER>@<IP_VPS>
# Check key login and sudo rights
ssh <USER>@<IP_VPS>
sudo -v
```

On Windows, PowerShell has `ssh-keygen` and `ssh`, but not `ssh-copy-id`. Copying the key in PowerShell:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <USER>@<IP_VPS> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

If the provider has already set up root login with a key and disabled password login, copy root's key to the new user:

```bash
# On the VPS as root
install -d -m 700 -o <USER> -g <USER> /home/<USER>/.ssh
install -m 600 -o <USER> -g <USER> /root/.ssh/authorized_keys /home/<USER>/.ssh/authorized_keys
```

**Expected:** `ssh <USER>@<IP_VPS>` logs in without the account password (it may only ask for the key passphrase), `sudo -v` accepts the user's password. From here on, all commands run as `<USER>` through `sudo`.

### 2.4. Key-only login

> **Important:** do this step only after the check in 2.3 has passed. Do not close the current SSH session until you have tested a new connection.

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null <<'EOF'
# SSH key login only, root login disabled
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
# Check syntax and apply
sudo sshd -t && sudo systemctl restart ssh
# Effective values
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

**Expected:**

```text
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
```

sshd reads the files in `/etc/ssh/sshd_config.d/` in alphabetical order and uses the first value it finds for each parameter. Cloud images sometimes ship `50-cloud-init.conf` with `PasswordAuthentication yes`, so this file name starts with `10-`: it is read first.

Check from your local computer in a new terminal window:

```bash
# Key login works
ssh <USER>@<IP_VPS> true && echo "key login works"
# Password login is rejected: expect "Permission denied (publickey)"
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password <USER>@<IP_VPS>
```

Rollback (if you lose access, use the provider's web console): `sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl restart ssh`.

### 2.5. Time

```bash
timedatectl
```

**Expected:** `System clock synchronized: yes` and `NTP service: active`. A wrong clock causes certificate verification errors. If synchronization is off:

```bash
sudo timedatectl set-ntp true
```

### 2.6. IP addresses and IPv6

```bash
# Addresses on network interfaces
ip -br addr
# Public IPv4 address as seen by external servers
curl -4 -s https://ipv4-internet.yandex.net/api/v0/ip; echo
# Public IPv6 address: an error or an empty response means IPv6 does not work
curl -6 -s --max-time 5 https://ipv6-internet.yandex.net/api/v0/ip; echo
```

- The public IPv4 address from the second command is present in the `ip -br addr` output - the VPS gets the address directly. This is the usual case.
- The interface has only a private address (`10.x.x.x`, `172.16-31.x.x`, `192.168.x.x`) - the provider uses NAT. coturn then needs the `external-ip` parameter (section 8.2).
- The third command returned an address - IPv6 works, and you can create an AAAA record for `xmpp.inkov.dev`. If there is no address, do not create an AAAA record.

### 2.7. Free ports

```bash
sudo ss -tulpn
sudo ss -tulpn | grep -E ':(443|5222|5269|5280|3478|5349)\b'
sudo ss -tulpn | grep -E ':50(0[0-9]{2}|100)\b'
```

**Expected:** the first output shows SSH on port 22 and the system DNS resolver on `127.0.0.53:53`, the two port checks print nothing. If the provider preinstalled a web server and port 443 is taken, stop it: for example, `sudo systemctl disable --now apache2` or `sudo systemctl disable --now nginx`.

### 2.8. Automatic security updates

```bash
systemctl is-enabled unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades
```

**Expected:** `enabled` and the line `APT::Periodic::Unattended-Upgrade "1";`. If the service is missing: `sudo apt install unattended-upgrades` and `sudo dpkg-reconfigure -plow unattended-upgrades`.

---

## 3. DNS records in Cloudflare

Run on: the Cloudflare dashboard in a browser. Checks run on the VPS or on a local computer.

### 3.1. Creating the records

1. Open https://dash.cloudflare.com and select the `inkov.dev` domain.
2. Go to **DNS → Records**.
3. For each row of the table, click **Add record**, fill in the fields and click **Save**.

| Type | Name | Value | Proxy status | TTL |
|---|---|---|---|---|
| A | `xmpp` | IPv4 address: `<IP_VPS>` | DNS only | Auto |
| CNAME | `conference` | Target: `xmpp.inkov.dev` | DNS only | Auto |
| CNAME | `upload` | Target: `xmpp.inkov.dev` | DNS only | Auto |
| CNAME | `turn` | Target: `xmpp.inkov.dev` | DNS only | Auto |
| SRV | `_xmpps-client._tcp` | Priority `0`, Weight `5`, Port `443`, Target `xmpp.inkov.dev` | n/a | Auto |
| SRV | `_xmpp-client._tcp` | Priority `10`, Weight `5`, Port `5222`, Target `xmpp.inkov.dev` | n/a | Auto |
| SRV | `_xmpp-server._tcp` | Priority `0`, Weight `5`, Port `5269`, Target `xmpp.inkov.dev` | n/a | Auto |
| AAAA, only if IPv6 works | `xmpp` | IPv6 address: address of the VPS | DNS only | Auto |
| CNAME, only for Proxy65 | `proxy` | Target: `xmpp.inkov.dev` | DNS only | Auto |

> **Important:** for A, AAAA and CNAME records Cloudflare turns on proxying by default (Proxy status: Proxied, orange cloud). Switch it to **DNS only** (grey cloud). The Cloudflare proxy passes only HTTP and HTTPS on a limited set of ports, XMPP and TURN do not work through it.

Notes on the fields:

- **Name** is relative to the zone: `xmpp` means `xmpp.inkov.dev`, `_xmpp-client._tcp` means `_xmpp-client._tcp.inkov.dev`. Cloudflare appends the zone name itself.
- SRV records are created in the `inkov.dev` zone, not on a subdomain: clients look them up by the domain in the user's address. If the SRV form shows separate Service and Protocol fields, set Service to `_xmpps-client` (or `_xmpp-client`, `_xmpp-server`), Protocol to `TCP`, Name to `@`.
- **Priority** sets the order in which targets are tried: lower values go first. Clients with XEP-0368 support merge the `_xmpps-client` and `_xmpp-client` records and connect to port 443 first (priority 0), then to 5222 on failure (priority 10). **Weight** spreads the load between servers with the same priority; for a single server any value works.
- The SRV record **Target** is a name with an A record (`xmpp.inkov.dev`), not a CNAME: RFC 2782 requires this.
- **TTL Auto** in Cloudflare is 300 seconds.
- For DNS only records the dashboard may warn that the IP address will be exposed. For this setup that is expected.
- Do not change the existing `inkov.dev` A records (GitHub Pages) or the `www` record, if there is one.

### 3.2. CAA records

CAA records limit the list of certificate authorities allowed to issue certificates for the domain.

```bash
# If dig is not found: sudo apt install -y bind9-dnsutils
dig +short CAA inkov.dev
```

- Empty output - there are no restrictions, nothing else to do.
- If there are records, one of them must be `0 issue "letsencrypt.org"`. If there are records with the `issuewild` tag, you also need `0 issuewild "letsencrypt.org"`, otherwise the wildcard certificate will not be issued. Add the missing record in Cloudflare: Type `CAA`, Name `@`, choose the tag in the **Tag** field, value `letsencrypt.org`.

GitHub Pages also gets its certificates from Let's Encrypt, so existing CAA records usually allow `letsencrypt.org` already.

### 3.3. Verification

Queries sent straight to the Cloudflare DNS server show the records right after they are saved, with no caching by intermediate resolvers.

```bash
# Cloudflare DNS server for the zone
NS=$(dig +short NS inkov.dev | head -1); echo "$NS"
dig +short @"$NS" A xmpp.inkov.dev
dig +short @"$NS" SRV _xmpps-client._tcp.inkov.dev
dig +short @"$NS" SRV _xmpp-client._tcp.inkov.dev
dig +short @"$NS" SRV _xmpp-server._tcp.inkov.dev
dig +short @"$NS" upload.inkov.dev
```

**Expected** (with the example address `203.0.113.10`):

```text
xxxx.ns.cloudflare.com.
203.0.113.10
0 5 443 xmpp.inkov.dev.
10 5 5222 xmpp.inkov.dev.
0 5 5269 xmpp.inkov.dev.
xmpp.inkov.dev.
203.0.113.10
```

Then check the answer from a regular resolver:

```bash
dig +short SRV _xmpps-client._tcp.inkov.dev
```

The answer must match the previous one. If it is empty, wait: a resolver that queried this name before the record was created keeps the negative answer for up to 30 minutes.

---

## 4. Installing Prosody

### 4.1. Prosody repository

```bash
# Repository reachable from the VPS: expect 200
curl -s -o /dev/null -w '%{http_code}\n' --max-time 15 http://packages.prosody.im/debian/dists/noble/Release
# Add the repository. The file contains the repository URL and the signing key (Signed-By),
# no separate key is needed
sudo wget https://prosody.im/downloads/repos/noble/prosody.sources -O /etc/apt/sources.list.d/prosody.sources
sudo apt update
# Version that will be installed
apt-cache policy prosody
```

**Expected output of** `apt-cache policy prosody`:

```text
prosody:
  Installed: (none)
  Candidate: 13.0.7-1~noble2
  Version table:
     13.0.7-1~noble2 500
        500 http://packages.prosody.im/debian noble/main amd64 Packages
     0.12.4-1build3 500
        500 http://archive.ubuntu.com/ubuntu noble/universe amd64 Packages
```

The version number may be newer. What matters is that `Candidate` comes from `packages.prosody.im` and is 13.0 or later.

If the repository is unreachable (curl returns `000` or times out), download the `prosody_13.*~noble*_amd64.deb` package on another computer from https://packages.prosody.im/debian/pool/main/p/prosody/, copy it to the VPS with `scp` and install it with `sudo apt install --no-install-recommends ./prosody_*.deb lua-unbound lua-readline`.

### 4.2. Installation

```bash
sudo apt install --no-install-recommends prosody lua-unbound lua-readline lua-dbi-sqlite3
```

> **Important:** the `--no-install-recommends` flag is required. Without it apt installs the recommended `luarocks` package. On Ubuntu 24.04 it depends on Lua 5.1 and switches the `lua` command to it, after which Prosody fails to start with the error `Prosody is no longer compatible with Lua 5.1`. For the same reason, do not install `luarocks`, `lua5.4` or `liblua5.4-dev` by hand: the required Lua 5.4 is installed as a package dependency.

Packages:

- `lua-dbi-sqlite3` is the LuaDBI driver for SQLite; Prosody works with the database through it. The `lua-sql-sqlite3` package (LuaSQL) is a different library that Prosody does not use, so do not install it.
- `lua-unbound` (DNS resolver) and `lua-readline` (line editing in `prosodyctl shell`) are recommended by Prosody and do not pull in extra dependencies.

Check:

```bash
sudo prosodyctl about | grep -E '^Prosody|Lua version|luaunbound'
# The SQLite driver loads in Lua 5.4
lua5.4 -e 'require "DBI"; print("LuaDBI: ok")'
systemctl status prosody --no-pager
```

**Expected:**

```text
Prosody 13.0.7
Lua version:             	Lua 5.4
luaunbound:   	1.0.0
LuaDBI: ok
```

and `Active: active (running)`. The error `module 'DBI' not found` means the driver is not installed or the OS is not Ubuntu 24.04. At this point Prosody runs with the default configuration (host `localhost`), which is replaced in section 7.

---

## 5. Firewall

On a clean server ufw is usually disabled. Add the rules first, then enable ufw.

> **Important:** the SSH rule is added before `ufw enable`. Do not close the current SSH session until you have tested a new connection.

```bash
# SSH: the OpenSSH profile opens 22/tcp
sudo ufw allow OpenSSH
# XMPP: Direct TLS and HTTPS, STARTTLS, server-to-server connections
sudo ufw allow 443/tcp comment 'XMPP direct TLS and HTTPS'
sudo ufw allow 5222/tcp comment 'XMPP clients'
sudo ufw allow 5269/tcp comment 'XMPP servers'
# List of added rules
sudo ufw show added
# Enable
sudo ufw enable
sudo ufw status verbose
```

**Expected output of** `ufw status verbose`:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
...
443/tcp                    ALLOW IN    Anywhere                   # XMPP direct TLS and HTTPS
```

`ufw enable` warns that existing SSH connections may be disrupted; confirm with `y`. Then open a new SSH connection. If you lose access, run `sudo ufw disable` in the provider's web console.

- If the provider set up SSH on a different port, allow that port instead of `OpenSSH`: `sudo ufw allow <PORT>/tcp`.
- Port 5280 is not opened: it is only needed locally. Port 80 is not needed: the certificate is issued through DNS.
- The coturn ports (3478 and 50000-50100/udp) are opened in section 8.3, after coturn is configured. Ports 5000 (Proxy65) and 5349 (TURN over TLS) are opened only if you enable those features (sections 7.3 and 8.4).
- The rules apply to both IPv4 and IPv6: `/etc/default/ufw` has `IPV6=yes` by default.
- If an external firewall is enabled in the provider's control panel, open the same ports there.

---

## 6. SSL certificate

### 6.1. Let's Encrypt reachability

```bash
curl -s -o /dev/null -w '%{http_code}\n' --max-time 15 https://acme-v02.api.letsencrypt.org/directory
```

**Expected:** `200`. Whether Let's Encrypt is reachable from Russia depends on the provider's network, so the guide relies on this check. If the result is `000` or another code, go to the fallback options in 6.8.

### 6.2. Cloudflare API token

certbot creates the temporary TXT record `_acme-challenge.inkov.dev` through the Cloudflare API. This requires a token that can edit DNS in the `inkov.dev` zone only.

Run on: the Cloudflare dashboard.

1. Open **My Profile → API Tokens** (https://dash.cloudflare.com/profile/api-tokens) and click **Create Token**.
2. Choose the **Edit zone DNS** template and click **Use template**.
3. **Permissions**: leave `Zone` / `DNS` / `Edit`.
4. **Zone Resources**: `Include` / `Specific zone` / `inkov.dev`.
5. **Client IP Address Filtering** (recommended): `Is in` and the address `<IP_VPS>`. If IPv6 works on the VPS, add the IPv6 address as well: API requests may go out over IPv6.
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

The token is pasted in the editor and does not end up in the shell history.

Check the token and the Cloudflare API reachability from the VPS (the token is read from the file):

```bash
sudo sh -c 'curl -s --max-time 20 https://api.cloudflare.com/client/v4/user/tokens/verify -H "Authorization: Bearer $(sed -n "s/^dns_cloudflare_api_token *= *//p" /root/.secrets/cloudflare.ini)"'; echo
```

**Expected:**

```text
{"result":{"id":"...","status":"active"},"success":true,"errors":[],"messages":[{"code":10000,"message":"This API Token is valid and active","type":null}]}
```

- `"success":false` - the token is wrong or was not copied completely.
- The command times out or returns an empty response - the Cloudflare API is unreachable from the provider's network. Since June 2025 Russian ISPs have been restricting connections to the Cloudflare network. In this case use the manual option from 6.8.

### 6.4. Installing certbot

```bash
sudo apt install certbot python3-certbot-dns-cloudflare
certbot --version
```

**Expected:** `certbot 2.9.0`. This is the version from the Ubuntu 24.04 repository; it supports DNS-01 through the Cloudflare API with a token. Do not install the snap version of certbot alongside it: both installations use the `/etc/letsencrypt` directory and run renewals independently of each other.

### 6.5. Test issuance

```bash
sudo certbot certonly --dry-run \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  -d inkov.dev -d '*.inkov.dev' \
  --email <EMAIL> --agree-tos --no-eff-email
```

Parameters:

- `--dry-run` - issuance against the Let's Encrypt staging server, without saving the certificate. It checks the token, DNS and Let's Encrypt reachability and does not count against the rate limit of 5 identical certificates per week.
- `--dns-cloudflare-propagation-seconds 60` - a pause before validation, so the TXT record reaches all Cloudflare DNS servers.
- `-d inkov.dev -d '*.inkov.dev'` - one certificate for the domain and all subdomains. The quotes around `*.inkov.dev` are required, otherwise the shell tries to expand it into file names.

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

`--deploy-hook` runs after every successful issuance and renewal. `prosodyctl --root cert import` copies the certificate and key to `/etc/prosody/certs` with the owner `prosody` and reloads Prosody. You cannot point Prosody directly at the files in `/etc/letsencrypt`: the key there is readable only by root, and Prosody runs as the `prosody` user.

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

**Expected:** `Domains: inkov.dev *.inkov.dev`, `Expiry Date` about 90 days ahead. `/etc/prosody/certs` contains `inkov.dev.crt` and `inkov.dev.key` owned by `prosody`.

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

With `--dry-run` the deploy hook does not run by default. Renewal together with the import command is tested in section 9.2, after Prosody is configured.

> **Important:** since June 2025 Let's Encrypt no longer sends certificate expiry emails. If renewal stops working (for example, the Cloudflare API becomes unreachable from the VPS), you will not get a notice. Check the expiry date once a month with `sudo certbot certificates` or set up external certificate monitoring for `https://upload.inkov.dev/`.

### 6.8. Fallback options

**Option 1. Manual DNS-01 challenge.** Use it if the Cloudflare API is unreachable from the VPS. The TXT records are created by hand in the Cloudflare dashboard, and there is no automatic renewal.

```bash
sudo certbot certonly --manual --preferred-challenges dns \
  -d inkov.dev -d '*.inkov.dev' \
  --email <EMAIL> --agree-tos --no-eff-email \
  --deploy-hook 'prosodyctl --root cert import /etc/letsencrypt/live'
```

certbot prints two values for the name `_acme-challenge.inkov.dev`, one after the other, one for each name in the certificate. For each value, create a record in Cloudflare: Type `TXT`, Name `_acme-challenge`, Content - the value from the output. Before pressing Enter the last time, make sure both records are visible:

```bash
dig +short @"$(dig +short NS inkov.dev | head -1)" TXT _acme-challenge.inkov.dev
```

After issuance, delete the TXT records. `certbot renew` does not work for such a certificate: repeat the command by hand before it expires, roughly every 60 days.

**Option 2. Another certificate authority.** Use it if Let's Encrypt is unreachable but the Cloudflare API works. ZeroSSL supports ACME and wildcard certificates, but requires EAB keys from the ZeroSSL account dashboard. Add these parameters to the command from 6.6:

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

In the Prosody 13 package the whole configuration is kept in a single file, `/etc/prosody/prosody.cfg.lua`. The `conf.avail` and `conf.d` directories from older Debian packages are not used.

### 7.2. TURN secret

```bash
sudo install -d -m 700 /root/.secrets
sudo sh -c 'umask 077; openssl rand -hex 32 > /root/.secrets/turn_secret'
```

The secret is inserted into the Prosody (7.4) and coturn (8.2) configurations with `sed`, there is no need to copy it by hand.

### 7.3. What the settings do

| Parameter | Value | Purpose |
|---|---|---|
| `admins` | `admin@inkov.dev`, `tzx1z@inkov.dev` | server administrators: commands from a client, invitations. The accounts are created separately (section 10.1) |
| `modules_enabled` | the list from the package, plus `mam`, `turn_external` and `net_multiplex` | server features, explained below |
| `ssl_ports` | `443` | TLS port on which `net_multiplex` accepts Direct TLS for clients and HTTPS |
| `https_ports` | empty list | a separate HTTPS port is not needed: HTTPS runs on 443 |
| `http_interfaces` | `127.0.0.1` | plain HTTP on port 5280 is reachable only from the server itself |
| `http_external_url` | `https://upload.inkov.dev/` (in the file sharing component) | URL used in file links. Without it Prosody puts the address of the unencrypted port 5280 there |
| `allow_registration` | `false` | registration from clients is closed. Accounts are created by the administrator or through invitations (the `invites*` modules) |
| `c2s_require_encryption`, `s2s_require_encryption` | `true` | connections without TLS are refused |
| `s2s_secure_auth` | `true` | other servers must present a valid certificate |
| `limits` | 10 and 30 KB/s | incoming traffic limits for clients and servers, values from the package |
| `pidfile` | `/var/run/prosody/prosody.pid` | `prosodyctl` uses it to reload the server, including after a certificate import |
| `authentication` | `internal_hashed` | passwords are stored as SCRAM hashes |
| `storage`, `sql` | `sql`, driver `SQLite3`, file `prosody.sqlite` | all Prosody data in one database, `/var/lib/prosody/prosody.sqlite`. Prosody creates the tables itself on first start. Uploaded files are stored on disk |
| `archive_expires_after` | `1y` | history is kept on the server for a year and synced across devices. A shorter period means less data on the server. Example values: `1w`, `30d`, `6 months`, `1y`, `never`. The value `1m` is ambiguous (month or minute), and Prosody 13 logs it as an error |
| `turn_external_*` | `turn.inkov.dev`, secret, TCP | clients get the coturn address and temporary credentials valid for 24 hours |
| `certificates` | `certs` | Prosody finds the certificate in `/etc/prosody/certs` itself, and for subdomains uses the wildcard |

How Prosody shares port 443: `net_multiplex` terminates TLS with its own certificate (chosen by SNI) and selects the service by ALPN: `xmpp-client` is a client connection, `http/1.1` is HTTPS. If the client sent no ALPN, the service is detected by the first bytes: an XMPP stream or an HTTP request. Server-to-server connections do not use 443 (section 1.3).

Modules that mobile clients depend on:

- `smacks` - resumes the session after a network change without losing messages;
- `carbons` - copies messages to all of the user's devices;
- `csi_simple` - delays low-priority traffic while the app is in the background;
- `cloud_notify` - push notifications (XEP-0357), shipped with Prosody 13 and enabled by default;
- `mam` - message archive on the server.

Components:

- `conference.inkov.dev` (`muc`) - group chats. `muc_mam` keeps room history, `restrict_room_creation = "local"` allows only `inkov.dev` users to create rooms.
- `upload.inkov.dev` (`http_file_share`) - file sharing. This is a separate component, not a module from `modules_enabled`. Limits: files up to 100 MB, up to 1 GB per user per day, up to 5 GB on the server, files kept for 30 days. Large files are streamed to disk, so `http_max_content_size` does not need to be changed.
- `proxy.inkov.dev` (`proxy65`) - direct file transfer for old clients. Modern clients use `http_file_share`, so the component is commented out. To enable it, remove `--` from the two lines, create the `proxy` DNS record (section 3.1) and open the port: `sudo ufw allow 5000/tcp comment 'XMPP proxy65'`.

### 7.4. Configuration file

Copy the whole block and run it in the terminal: the command replaces the contents of the file.

```bash
sudo tee /etc/prosody/prosody.cfg.lua > /dev/null <<'EOF'
-- /etc/prosody/prosody.cfg.lua
-- Personal XMPP server for addresses like user@inkov.dev.
-- Based on the configuration from the Prosody 13.0 package. Clean Ubuntu 24.04 server.
-- After every change: sudo prosodyctl check config

---------- Global settings ----------
-- Apply to the whole server. Must come before the first VirtualHost or Component line.

-- Administrators. The accounts have to be created separately with prosodyctl adduser.
admins = { "admin@inkov.dev", "tzx1z@inkov.dev" }

modules_enabled = {
    -- Required
        "disco"; -- discovery of server features
        "roster"; -- contact list
        "saslauth"; -- authentication
        "tls"; -- connection encryption

    -- Recommended
        "blocklist"; -- blocking users
        "bookmarks"; -- syncs the list of group chats between clients
        "carbons"; -- copies messages to all of the user's devices
        "dialback"; -- fallback verification of other servers through DNS
        "limits"; -- rate limiting for incoming connections
        "pep"; -- account data: avatars, OMEMO keys
        "private"; -- legacy storage for client settings (XEP-0049)
        "smacks"; -- resumes connections without losing messages (XEP-0198)
        "vcard4"; -- user profiles
        "vcard_legacy"; -- compatibility with the old profile format

    -- Mobile clients and convenience
        "account_activity"; -- time of the last login to the account
        "cloud_notify"; -- push notifications for mobile clients (XEP-0357)
        "csi_simple"; -- saves traffic and battery on phones
        "invites"; -- invitations
        "invites_adhoc"; -- creating invitations from a client
        "invites_register"; -- registration by invitation while registration is closed
        "ping"; -- replies to XMPP ping
        "register"; -- password change from a client; allow_registration keeps registration closed
        "time"; -- server time
        "uptime"; -- server uptime
        "version"; -- server version
        "mam"; -- message archive on the server (XEP-0313)
        "turn_external"; -- TURN server details for calls (XEP-0215)

    -- Administration
        "admin_adhoc"; -- administrator commands from a client
        "admin_shell"; -- the sudo prosodyctl shell console

    -- Network
        "net_multiplex"; -- several services on one port (443)
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

-- Required by prosodyctl: it uses this file to find the process to reload,
-- including after a certificate update
pidfile = "/var/run/prosody/prosody.pid"

-- Passwords are stored as hashes (SCRAM)
authentication = "internal_hashed"

-- Storage: an SQLite database in a single file. Requires the lua-dbi-sqlite3 driver (section 4.2).
-- Prosody creates the tables itself on first start.
storage = "sql"
sql = {
    driver = "SQLite3";
    database = "prosody.sqlite"; -- path relative to /var/lib/prosody
}

-- Retention period for the archive of one-to-one messages
archive_expires_after = "1y"

-- Port 443: Direct TLS for clients (XEP-0368) and HTTPS for file sharing.
-- Prosody selects the service by ALPN, or by the first bytes if the client sent no ALPN.
ssl_ports = { 443 }
-- A separate HTTPS port is not needed: HTTPS runs on port 443
https_ports = { }
-- Plain HTTP (5280) is reachable only from the server itself
http_ports = { 5280 }
http_interfaces = { "127.0.0.1" }

-- TURN server for calls (coturn). The secret matches static-auth-secret in /etc/turnserver.conf.
-- If you do not need calls, delete these three lines and "turn_external" from modules_enabled.
turn_external_host = "turn.inkov.dev"
turn_external_secret = "<TURN_SECRET>"
turn_external_tcp = true

-- Logs
log = {
    info = "/var/log/prosody/prosody.log"; -- change info to debug for a detailed log
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

-- File sharing: https://upload.inkov.dev/ (port 443)
Component "upload.inkov.dev" "http_file_share"
    http_external_url = "https://upload.inkov.dev/" -- URL used in file links
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

Insert the secret, set permissions and check the file:

```bash
# TURN secret from the file in place of <TURN_SECRET>
sudo sh -c 'sed -i "s/<TURN_SECRET>/$(cat /root/.secrets/turn_secret)/" /etc/prosody/prosody.cfg.lua'
# No placeholders left: expect 0
sudo grep -c '<TURN_SECRET>' /etc/prosody/prosody.cfg.lua
# Owner root, group prosody, no access for others
sudo chown root:prosody /etc/prosody/prosody.cfg.lua
sudo chmod 640 /etc/prosody/prosody.cfg.lua
# Check syntax and parameters
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

The message about `check features` refers to browser connections (section 9.3) and does not affect the server. The new configuration takes effect on the restart in section 9.1.

---

## 8. Calls: coturn

If you do not need calls, skip this section. In that case delete the `"turn_external";` line and the three `turn_external_*` lines from `/etc/prosody/prosody.cfg.lua`.

### 8.1. Installation

```bash
sudo apt install coturn
# The service starts right after installation with the configuration file from the package.
# Stop it until it is configured.
sudo systemctl stop coturn
```

> **Important:** with the configuration file from the package, coturn allocates relay ports without checking credentials. That is why the service is stopped right after installation, and the TURN ports are opened in the firewall only after it is configured (section 8.3).

### 8.2. Configuration

```bash
# Backup of the file from the package
sudo cp /etc/turnserver.conf /etc/turnserver.conf.orig
sudo tee /etc/turnserver.conf > /dev/null <<'EOF'
# /etc/turnserver.conf - TURN server for XMPP calls

# STUN/TURN port (UDP and TCP)
listening-port=3478

# UDP port range for media relay. Must match the firewall rule.
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

# Do not relay to local, private and special-purpose addresses.
# Without this, TURN can be used to reach services that listen only on 127.0.0.1.
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
# TURN secret from the file in place of <TURN_SECRET>
sudo sh -c 'sed -i "s/<TURN_SECRET>/$(cat /root/.secrets/turn_secret)/" /etc/turnserver.conf'
# Only root and the turnserver group can read the file
sudo chown root:turnserver /etc/turnserver.conf
sudo chmod 640 /etc/turnserver.conf
```

If section 2.6 showed that the VPS is behind NAT, uncomment the `external-ip` line and set the public and internal addresses: `sudo nano /etc/turnserver.conf`.

What the parameters do:

- `use-auth-secret` and `static-auth-secret` - Prosody gives clients a temporary username and password derived from the shared secret. Permanent TURN passwords are not needed.
- `min-port` and `max-port` - 101 ports for media. One call through TURN takes several ports, which is enough for a personal server.
- `no-cli` disables the coturn management console, `no-software-attribute` hides the coturn version in responses.
- `denied-peer-ip` - blocks relaying to internal addresses. Without it a TURN user can reach services that listen only on `127.0.0.1` or on the provider's internal network.

### 8.3. Startup and verification

```bash
sudo systemctl start coturn
# Start on boot: expect enabled
systemctl is-enabled coturn
systemctl status coturn --no-pager
sudo ss -tulpn | grep turnserver
```

**Expected:** `Active: active (running)`, `turnserver` listens on port 3478 over UDP and TCP.

TURN ports in the firewall: STUN/TURN over TCP and UDP, and the media relay range.

```bash
sudo ufw allow 3478 comment 'STUN/TURN'
sudo ufw allow 50000:50100/udp comment 'TURN relay'
sudo ufw status | grep -E '3478|50000'
```

If the provider has an external firewall, open the same ports there.

Check that Prosody and coturn work together. The command reads the `turn_external_*` settings from the Prosody configuration, so Prosody does not need a restart for it. The name `turn.inkov.dev` must already resolve (section 3).

```bash
# Credentials issued and a relay port allocated
sudo prosodyctl check turn -v
# Same, plus a packet relayed to an external STUN server
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

The port in `Relayed address` is within the 50000-50100 range. The warning `STUN returned a private IP! Is the TURN server behind a NAT and misconfigured?` means the VPS is behind NAT and the `external-ip` parameter is needed.

### 8.4. TLS for coturn (optional)

TURN over TLS on port 5349 helps clients on networks where UDP is blocked but TCP connections with TLS are allowed. Port 443 cannot be used for this: Prosody holds it.

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
# coturn: remove the TLS restriction, add the port and certificate
sudo sed -i -e '/^no-tls$/d' -e '/^no-dtls$/d' /etc/turnserver.conf
sudo tee -a /etc/turnserver.conf > /dev/null <<'EOF'

# TLS (section 8.4)
tls-listening-port=5349
cert=/etc/coturn/certs/fullchain.pem
pkey=/etc/coturn/certs/privkey.pem
no-tlsv1
no-tlsv1_1
EOF
# Prosody: advertise the TLS TURN port to clients
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

**Expected:** a line with the expiry date and a line with `DNS:inkov.dev` and `DNS:*.inkov.dev`. On every certificate renewal (about every 60 days) coturn restarts, and calls going through TURN at that moment are dropped.

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
portmanager	info	Activated service 'http' on [127.0.0.1]:5280
portmanager	info	Activated service 'https' on no ports
upload.inkov.dev:http	info	Serving 'file_share' at https://upload.inkov.dev/file_share
portmanager	info	Activated service 's2s' on [::]:5269, [*]:5269
portmanager	info	Activated service 'multiplex_ssl' on [::]:443, [*]:443
portmanager	info	Activated service 'c2s' on [::]:5222, [*]:5222
```

Port 443 is served by `multiplex_ssl`, and the line `'https' on no ports` is expected: HTTPS runs on 443. `prosody.err` is empty or does not exist. If the service did not start, `journalctl -u prosody -n 50 --no-pager` shows the reason.

Check the SQLite database:

```bash
sudo ls -l /var/lib/prosody/prosody.sqlite
sudo grep -c -E 'LuaDBI or LuaSQLite3|no data storage' /var/log/prosody/prosody.err
```

**Expected:** the `prosody.sqlite` file owned by `prosody` with permissions `-rw-r-----`, error count `0`. If the file is missing or the count is above zero, Prosody is running without storage: check the driver (section 4.2).

### 9.2. Certificate import and renewal test

certbot runs this same command after every renewal. The configuration now has the `inkov.dev` host and the components, so running it by hand works as well:

```bash
sudo prosodyctl --root cert import /etc/letsencrypt/live
```

**Expected:** `Imported certificate and key for hosts inkov.dev, upload.inkov.dev, conference.inkov.dev`. Prosody reloads the certificates automatically, and `Certificates reloaded` appears in the log.

Renewal test together with the import command (certbot 2.x has the `--run-deploy-hooks` flag):

```bash
sudo certbot renew --dry-run --run-deploy-hooks
```

**Expected:** `Congratulations, all simulated renewals succeeded` and the import command message `Imported certificate and key for hosts ...`.

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
| `check config` | configuration syntax and parameters | `All checks passed, congratulations!` |
| `check dns` | SRV records, A and AAAA records of the server and components, whether the addresses match the VPS IP | only the expected messages about `_xmpps-server` (see below) |
| `check certs` | certificates for `inkov.dev` and the components, names and validity | `Certificate: /etc/prosody/certs/inkov.dev.crt` for each host |
| `check turn -v` | TURN credential issuance and relay port allocation | `Success!` |
| `check connectivity` | reachability of ports 5222 and 5269 from the internet, through the observe.jabber.network service | `xmpp-client: Works`, `xmpp-server: Works` |
| `check features` | the feature set for clients | every item `OK` except `Web connections` |

Specifics of this setup:

- With `net_multiplex`, `check dns` assumes that port 443 also serves server-to-server connections and prints `No _xmpps-server SRV record found for inkov.dev, but it looks like you need one.`, plus the same lines for `conference.inkov.dev` and `upload.inkov.dev`. These messages are expected: server-to-server connections go over 5269 (section 1.3). There should be no other messages.
- `check dns` checks the SRV targets. The message `inkov.dev A record points to unknown address 185.199.x.x` means the server cannot see the SRV records: go back to section 3.3.
- `check connectivity` does not check port 443, that check is in section 9.4. If the external service is unreachable from the provider's network, the message `Failed to request check at API` does not mean there is a problem with the server.
- `check features` flags `(!) Web connections`: these are BOSH and WebSocket for browser clients. Mobile and desktop clients do not need them.
- If the VPS is behind NAT, `check dns` reports that the addresses from DNS are not found on the server. Set the public address in the global part of the configuration, above the `VirtualHost` line: `external_addresses = { "<IP_VPS>" }`.

### 9.4. Checking from outside

Run on: a local computer with Linux or macOS. On Windows, check ports in PowerShell: `Test-NetConnection xmpp.inkov.dev -Port 443`, expect `TcpTestSucceeded : True`.

```bash
# SRV records through a regular resolver
dig +short SRV _xmpps-client._tcp.inkov.dev
dig +short SRV _xmpp-client._tcp.inkov.dev
dig +short SRV _xmpp-server._tcp.inkov.dev

# Port reachability
nc -vz xmpp.inkov.dev 443
nc -vz xmpp.inkov.dev 5222
nc -vz xmpp.inkov.dev 5269

# Direct TLS on 443: Prosody replies with the list of login mechanisms
(printf "<?xml version='1.0'?><stream:stream to='inkov.dev' xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' version='1.0'>"; sleep 3) \
  | timeout 6 openssl s_client -connect xmpp.inkov.dev:443 -servername inkov.dev -alpn xmpp-client -quiet 2>/dev/null \
  | grep -o '<mechanism>[A-Z0-9-]*</mechanism>'
# HTTPS for file sharing on 443
curl -s -o /dev/null -w '%{http_code}\n' https://upload.inkov.dev/file_share/
# Certificates: must include inkov.dev
openssl s_client -connect xmpp.inkov.dev:443 -servername inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
openssl s_client -connect xmpp.inkov.dev:5222 -starttls xmpp -xmpphost inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
openssl s_client -connect xmpp.inkov.dev:5269 -starttls xmpp-server -xmpphost inkov.dev </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate -text | grep -E 'notAfter=|DNS:'
```

**Expected:**

- SRV records as in section 3.3;
- `nc` reports a successful connection (`Connected to` or `succeeded`) for all three ports;
- lines like `<mechanism>SCRAM-SHA-1-PLUS</mechanism>`;
- `curl` prints `200`;
- for each port, the certificate expiry date and a line with `DNS:inkov.dev` and `DNS:*.inkov.dev`.

### 9.5. External test services

- https://compliance.conversations.im checks support for the XMPP extensions that matter for mobile clients. The service logs in to the server with an account, so use a temporary `test@inkov.dev` (section 10.1) and delete it after the test.
- Whether external services are reachable from Russia and from the provider's network can change. The main checks are the commands from 9.3 and 9.4.

---

## 10. Users and clients

### 10.1. Accounts

```bash
# Administrators (addresses listed in admins). The password is typed twice and does not end up in the shell history
sudo prosodyctl adduser admin@inkov.dev
sudo prosodyctl adduser tzx1z@inkov.dev
# Test user
sudo prosodyctl adduser test@inkov.dev
# List of users
sudo prosodyctl shell user list inkov.dev
# Effective administrator role
sudo prosodyctl shell user role admin@inkov.dev
```

**Expected:** `OK: Created admin@inkov.dev with role 'prosody:member'` - the role at creation time. Administrator rights come from the `admins` list, so `user role` shows `OK: prosody:operator`. Accounts are stored in the `/var/lib/prosody/prosody.sqlite` database. The error `Could not create user: no data storage active` means storage is not working (section 12).

Other operations:

```bash
# Change the password
sudo prosodyctl passwd test@inkov.dev
# Delete the account
sudo prosodyctl deluser test@inkov.dev
```

Invitation (optional). The person sets the password themselves, and the administrator does not need to know it:

```bash
sudo prosodyctl shell invite create_account friend@inkov.dev
```

**Expected:** `OK: xmpp:friend@inkov.dev?register;preauth=...`. The link opens in Conversations and clients based on it (Monocles Chat, Cheogram): they create an account with a password chosen by the user. The invitation lifetime is set with the `--expires-after` flag.

### 10.2. Clients

| Platform | Client | Where to get it |
|---|---|---|
| Android | Conversations | F-Droid: https://f-droid.org/packages/eu.siacs.conversations/, website https://conversations.im |
| Android | Monocles Chat | F-Droid: https://f-droid.org/packages/de.monocles.chat/ |
| Android | Cheogram | https://cheogram.com |
| iOS, macOS | Monal | App Store, website https://monal-im.org |
| Linux | Dino | https://dino.im, distribution packages or Flathub |
| Linux, Windows | Gajim | https://gajim.org |

Conversations is a paid app on Google Play, and paying for Google Play purchases from Russia has not been possible since 2022. The F-Droid version is free.

### 10.3. Connecting a client

- Address: `user@inkov.dev`, password from `prosodyctl adduser`.
- There is no need to enter the server and port: the client finds them through the SRV records. Clients with XEP-0368 support connect to port 443, others to 5222.
- If the client cannot find the server (for example, DNS on the network does not answer SRV queries), set the server `xmpp.inkov.dev` and port `5222` by hand in the account's advanced settings. Use port 443 in a manual setup only if the client has a Direct TLS option: there is no STARTTLS on port 443.

### 10.4. Testing features

1. Log in as `admin@inkov.dev` on a phone and as `test@inkov.dev` on a computer.
2. Add each other as contacts and exchange messages.
3. Check OMEMO encryption: Conversations turns it on by default, in Gajim and Dino it is enabled in the chat window.
4. Send a photo: the file link starts with `https://upload.inkov.dev/file_share/`, with no port number.
5. Create a group chat: its address looks like `room@conference.inkov.dev`.
6. Make a call between devices on different networks, for example mobile data and Wi-Fi: this tests TURN.
7. Add a contact from another XMPP server and exchange messages: this tests federation.
8. Delete the test user: `sudo prosodyctl deluser test@inkov.dev`.

---

## 11. Security and maintenance

### 11.1. Access

- Login to the server is by SSH key only, root login is disabled (section 2.4).
- Registration is closed (`allow_registration = false`), only the administrator creates accounts.
- Use different long passwords for XMPP and sudo.
- Port 5280 and the `prosodyctl shell` console are available only locally, do not expose them.
- The Cloudflare token can change DNS in the `inkov.dev` zone. If you suspect a leak, delete it in **My Profile → API Tokens**, create a new one and update `/root/.secrets/cloudflare.ini`.

### 11.2. Updates

- Ubuntu security updates are installed by `unattended-upgrades` (section 2.8).
- By default `unattended-upgrades` does not update packages from the Prosody repository. Once a month run:

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
```

- The `prosody` package follows the current stable release, including the move to the next major version. Before such an upgrade, read the release notes at https://prosody.im. To update Prosody only by hand: `sudo apt-mark hold prosody`; to remove the hold: `sudo apt-mark unhold prosody`.

### 11.3. Backups

Do not copy the database file with `cp` or `tar` while Prosody is running: the copy may be inconsistent. The `sqlite3` utility takes a consistent copy without stopping the server.

```bash
# sqlite3 utility (once)
sudo apt install sqlite3
# Consistent database copy on a running server
sudo sqlite3 /var/lib/prosody/prosody.sqlite ".backup /root/prosody-$(date +%F).sqlite"
# Check the copy: expect ok
sudo sqlite3 /root/prosody-$(date +%F).sqlite "PRAGMA integrity_check;"
# Archive: database copy, configuration, certificates and secrets. Files uploaded by users are not included
sudo tar -czf /root/xmpp-backup-$(date +%F).tar.gz \
  /root/prosody-$(date +%F).sqlite /etc/prosody /etc/letsencrypt /etc/turnserver.conf /root/.secrets
sudo ls -lh /root/xmpp-backup-*.tar.gz
```

The message `tar: Removing leading '/' from member names` is expected. Copying the archive to your local computer:

```bash
# On the VPS: copy the archive to your home directory, readable only by you
sudo install -m 600 -o "$USER" /root/xmpp-backup-$(date +%F).tar.gz ~/
# On the local computer
scp <USER>@xmpp.inkov.dev:~/xmpp-backup-*.tar.gz .
# On the VPS: delete the copy from your home directory
rm ~/xmpp-backup-*.tar.gz
```

Restore:

```bash
# Configuration files, certificates, secrets and the database copy in /root
sudo tar -xzf xmpp-backup-<date>.tar.gz -C /
# The database is replaced while Prosody is stopped
sudo systemctl stop prosody
sudo install -o prosody -g prosody -m 640 /root/prosody-<date>.sqlite /var/lib/prosody/prosody.sqlite
sudo systemctl start prosody
sudo systemctl restart coturn
sudo prosodyctl shell user list inkov.dev
```

> **Important:** the archive contains the database with accounts and message history, the certificate private key, the Cloudflare token and the TURN secret. Store the copy encrypted.

### 11.4. Logs

- Prosody: `/var/log/prosody/prosody.log` and `prosody.err`. The package sets up rotation: daily, 14 files.
- coturn: `journalctl -u coturn`.
- certbot: `/var/log/letsencrypt/letsencrypt.log`.
- SSH: `journalctl -u ssh`.

### 11.5. fail2ban

Optional. SSH login is possible only with a key, so SSH password guessing is not possible. By default Prosody 13 does not log the IP addresses of failed login attempts, so fail2ban needs the third-party module `mod_log_auth` (https://modules.prosody.im/mod_log_auth). With long random passwords brute force is not effective, so fail2ban is optional for a personal server.

### 11.6. OS support period

Standard support for Ubuntu 24.04 LTS lasts until 2029. When moving to the next LTS release:

1. Make a backup (11.3).
2. `do-release-upgrade` disables third-party repositories. After the upgrade, add the Prosody repository for the new release:

```bash
sudo wget https://prosody.im/downloads/repos/$(lsb_release -sc)/prosody.sources -O /etc/apt/sources.list.d/prosody.sources
sudo apt update && sudo apt upgrade
```

3. Check the services: `sudo prosodyctl check`, `systemctl status coturn --no-pager`, `sudo certbot renew --dry-run`.

---

## 12. Troubleshooting

| Symptom | Likely cause | How to check | How to fix |
|---|---|---|---|
| Prosody does not start, the log shows `Prosody is no longer compatible with Lua 5.1` | `luarocks` or `lua5.1` is installed, the `lua` command points to Lua 5.1 | `readlink -f /usr/bin/lua` | `sudo update-alternatives --set lua-interpreter /usr/bin/lua5.4`, then `sudo systemctl restart prosody` |
| `prosody.err` contains `LuaDBI or LuaSQLite3 are required for using SQL databases`, `prosodyctl adduser` reports `no data storage active`, login does not work | `lua-dbi-sqlite3` is not installed or the OS is not Ubuntu 24.04 (on 22.04 the driver is not built for Lua 5.4) | `lua5.4 -e 'require "DBI"'`, `lsb_release -ds` | `sudo apt install --no-install-recommends lua-dbi-sqlite3`, then `sudo systemctl restart prosody`. On Ubuntu 22.04 use [GUIDE_XMPP_Prosody_Ubuntu_22.04_HAProxy_3x-ui.en.md](GUIDE_XMPP_Prosody_Ubuntu_22.04_HAProxy_3x-ui.en.md) with the default storage |
| `prosody.err` grows fast and contains long `stack traceback` entries | Most often storage is not working (see the row above) | `sudo grep -m 5 -v -E '^\s' /var/log/prosody/prosody.err` | Fix the first error that comes before the traceback |
| Prosody does not start after a configuration change | Lua syntax error: a missing quote, bracket or `;` | `sudo prosodyctl check config` shows the line with the error | Fix the line or restore `/etc/prosody/prosody.cfg.lua.orig` |
| The log shows a bind error on port 443 (`Permission denied`) | Prosody was started by hand, not through systemd. The systemd service gives Prosody the right to listen on ports below 1024 | `systemctl status prosody --no-pager` | Stop the manually started process and use `sudo systemctl start prosody` |
| The log shows `Address already in use` | The port is taken by another process | `sudo ss -tulpn`, find the port in the output | Stop that process (for example, a preinstalled web server on 443) |
| The client cannot find the server | SRV records are not created or not yet visible to the resolver; ports 443 and 5222 are closed | `dig +short SRV _xmpps-client._tcp.inkov.dev`, `nc -vz xmpp.inkov.dev 443` | Check the records (section 3), the VPS firewall and the provider's firewall |
| The client connects over 5222 but not over 443 | No `_xmpps-client` record, the client does not support XEP-0368, or port 443 is closed | Checks from 9.4 | Create the record, open the port. Clients without XEP-0368 connect to 5222 |
| The client reports a certificate error | The certificate does not include `inkov.dev`, was not imported, or has expired | The `openssl` commands from 9.4, `sudo prosodyctl check certs` | Issue the certificate with `-d inkov.dev -d '*.inkov.dev'`, run `sudo prosodyctl --root cert import /etc/letsencrypt/live` |
| certbot: `Error determining zone_id: 9109 ... Did you enter a valid Cloudflare Token?` | Wrong token | The token check from 6.3 | Copy the token again or create a new one |
| certbot: `Unable to determine zone_id for inkov.dev` | The token has no access to the zone | Token settings in Cloudflare | Zone Resources: `Include` / `Specific zone` / `inkov.dev` |
| certbot: `Error communicating with the Cloudflare API` with a hint about `Zone:DNS:Edit` | The token has no permission to edit DNS | Token permissions | `Zone` / `DNS` / `Edit` |
| certbot or the token check times out | The Cloudflare API is unreachable from the provider's network | The token check from 6.3 | Manual DNS-01 challenge (6.8, option 1) |
| certbot: `DNS problem: NXDOMAIN looking up TXT` or `Incorrect TXT record` | The TXT record did not propagate in time | Run the command again | Increase `--dns-cloudflare-propagation-seconds` to 120 |
| Messages to other servers are not delivered | Port 5269 is closed, no `_xmpp-server` SRV record, the remote server has an invalid certificate | `nc -vz xmpp.inkov.dev 5269`, `sudo tail -n 50 /var/log/prosody/prosody.log` | Open the port, check the SRV record; for a single domain with an invalid certificate, add `s2s_insecure_domains = { "example.org" }` to the global part of the configuration |
| Files are not sent | Port 443 is closed, no `upload` record, a limit is exceeded | `nc -vz xmpp.inkov.dev 443`, `curl` from 9.4 | Open the port, create the record, change the limits in the `upload.inkov.dev` component |
| File links contain `:5280` or `http://` | The `upload.inkov.dev` component has no `http_external_url` | The `Serving 'file_share' at` line in the Prosody log | Add `http_external_url = "https://upload.inkov.dev/"` to the component and restart Prosody |
| Calls do not connect or there is no audio | coturn is stopped, the 50000-50100/udp range is closed, the secrets do not match, the VPS is behind NAT without `external-ip` | `systemctl status coturn`, `sudo prosodyctl check turn -v --ping=stun.conversations.im` | Start coturn, open the ports, insert the secret again, set `external-ip` |
| `check turn`: `STUN returned a private IP` | The VPS is behind NAT | `ip -br addr` | `external-ip=<IP_VPS>/<INTERNAL_IP>` in `/etc/turnserver.conf`, then `sudo systemctl restart coturn` |
| Prosody: `Permission denied` when loading the key | The configuration has a path to the key in `/etc/letsencrypt` | `sudo cat /var/log/prosody/prosody.err` | Remove the explicit certificate paths, run `cert import` |
| SSH login fails after section 2.4 | The key was not copied, or login was tested as a different user | Provider's web console: `ls -l /home/<USER>/.ssh/authorized_keys` | In the provider's web console: `sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl restart ssh`, then repeat section 2.3 |
| SSH access is lost after enabling ufw | No SSH rule was added, or SSH runs on a different port | Provider's web console: `sudo ufw status numbered` | `sudo ufw allow OpenSSH` or `sudo ufw allow <PORT>/tcp`, or `sudo ufw disable` |
| `check dns`: `No _xmpps-server SRV record found ..., but it looks like you need one.` | A quirk of the check when `net_multiplex` is used | Section 9.3 | Nothing to do: server-to-server connections go over 5269 |
| `check dns`: `inkov.dev A record points to unknown address 185.199.x.x` | The server's resolver cannot see the SRV records | `dig +short SRV _xmpp-client._tcp.inkov.dev` on the VPS | Check the SRV records in Cloudflare, wait up to 30 minutes |

Detailed Prosody log: in `/etc/prosody/prosody.cfg.lua` change `info =` to `debug =` in the `log` block, run `sudo systemctl restart prosody`, reproduce the problem and switch back to `info`.

---

## 13. Summary

1. Prosody 13 on a clean Ubuntu 24.04 VPS serves `user@inkov.dev` addresses, while the `inkov.dev` site stays on GitHub Pages. Data is stored in the SQLite database `/var/lib/prosody/prosody.sqlite`.
2. Clients connect over port 443 (Direct TLS) or 5222, other servers over 5269. SRV records in the `inkov.dev` zone point to `xmpp.inkov.dev`.
3. File sharing works over HTTPS on port 443, and file links have no port number. Calls go through coturn.
4. The certificate for `inkov.dev` and `*.inkov.dev` is issued with DNS-01 and the Cloudflare API, renewed automatically and imported into Prosody.
5. Login to the server is by SSH key only, the firewall opens only the required ports, registration on the XMPP server is closed, and coturn does not relay traffic to internal addresses.

### Checklist

- [ ] OS updated, the sudo user logs in with a key, password and root login disabled (section 2)
- [ ] DNS records created in DNS only mode and visible with `dig` (section 3)
- [ ] Prosody 13 and `lua-dbi-sqlite3` installed with `--no-install-recommends`, `prosodyctl about` shows Lua 5.4, `LuaDBI: ok` (section 4)
- [ ] `/var/lib/prosody/prosody.sqlite` created after start, no storage errors in `prosody.err` (section 9.1)
- [ ] ufw enabled, SSH works (section 5)
- [ ] Certificate issued, `sudo certbot renew --dry-run` passes (section 6)
- [ ] `sudo prosodyctl check config` shows no errors (section 7)
- [ ] `sudo prosodyctl check turn -v` shows `Success!` (section 8)
- [ ] Port 443 answers for XMPP and HTTPS, `check certs` and `check connectivity` show no errors (section 9)
- [ ] Messages, files, group chats, calls and federation work (section 10)
- [ ] Backup created and copied off the server (section 11)

### Security measures

The server is reachable from the internet, so its security depends on regular maintenance. Once a month update the packages and check the certificate expiry date (`sudo certbot certificates`). Do not expose port 5280, do not enable password login for SSH, and do not disable `s2s_secure_auth` and `c2s_require_encryption`. Keep the SSH key, the files from `/root/.secrets` and the backups in a safe place.

### Documentation

- Prosody: https://prosody.im/doc
- Prosody package repository: https://prosody.im/download/package_repository
- Configuration: https://prosody.im/doc/configure
- Port multiplexing: https://prosody.im/doc/modules/mod_net_multiplex
- Certificates: https://prosody.im/doc/certificates, Let's Encrypt: https://prosody.im/doc/letsencrypt
- DNS: https://prosody.im/doc/dns
- TURN: https://prosody.im/doc/turn
- mod_http_file_share: https://prosody.im/doc/modules/mod_http_file_share
- mod_turn_external: https://prosody.im/doc/modules/mod_turn_external
- mod_mam: https://prosody.im/doc/modules/mod_mam
- mod_muc: https://prosody.im/doc/modules/mod_muc
- prosodyctl: https://prosody.im/doc/prosodyctl
- Dependencies and Lua versions: https://prosody.im/doc/depends
- Data storage: https://prosody.im/doc/storage
- mod_storage_sql: https://prosody.im/doc/modules/mod_storage_sql
- XEP-0368 (Direct TLS): https://xmpp.org/extensions/xep-0368.html
- coturn: https://github.com/coturn/coturn
- certbot: https://eff-certbot.readthedocs.io/en/stable/
- certbot plugin for Cloudflare: https://certbot-dns-cloudflare.readthedocs.io/en/stable/
- DNS in Cloudflare: https://developers.cloudflare.com/dns/
- Cloudflare API tokens: https://developers.cloudflare.com/fundamentals/api/get-started/create-token/
