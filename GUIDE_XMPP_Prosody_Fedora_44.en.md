# Step-by-step guide: Prosody XMPP server on a clean VPS in Russia (Fedora Server 44)

Versions: Fedora Server 44, Prosody 13.0.6, SQLite driver from the lua-dbi 0.7.5 package, coturn 4.18.0 and certbot 5.8.0 with the dns-cloudflare plugin. All packages come from the Fedora repositories. SELinux runs in enforcing mode. DNS for the domain is hosted on Cloudflare. The Prosody and coturn configurations, the certificate import and the SASL2, Bind 2 and FAST modules (section 14) were tested on these versions.

This guide is for a dedicated server with no other services on it. The layout is the same as in [GUIDE_XMPP_Prosody_Ubuntu_26.04.en.md](GUIDE_XMPP_Prosody_Ubuntu_26.04.en.md): Prosody holds port 443, data is stored in SQLite, calls go through coturn. What is different on Fedora:

- Prosody is installed from the Fedora repository. This is the package the Prosody project recommends for Fedora; the packages.prosody.im repository is meant for Debian and Ubuntu. Fedora ships only Lua 5.4, so the Lua version conflict described for Ubuntu does not happen here.
- SELinux runs in enforcing mode. To let Prosody listen on port 443, the `prosody_bind_http_port` policy boolean is turned on (section 4.3). There is no need to disable SELinux.
- On Fedora the `prosody` service runs as the `prosody` user without the right to listen on ports below 1024. The right is added with a systemd drop-in file (section 4.3).
- The firewall is firewalld, not ufw (section 5).
- Prosody certificates are stored in `/etc/pki/prosody`, the coturn configuration is in `/etc/coturn/turnserver.conf`, and the coturn service runs as the `coturn` user.
- Services do not start right after the packages are installed. The certificate renewal timer `certbot-renew.timer` is also enabled by hand (section 6.7).
- A Fedora release is supported for about 13 months. Fedora 44 gets updates until June 2027, after that the server has to be upgraded to the next release (section 11.6).
- Section 14 (optional) covers SASL2, Bind 2 and FAST: faster reconnection for mobile clients.

> **Important:** this guide was tested on Fedora Server. Fedora cloud images (Cloud Edition) may not include firewalld and Cockpit. The differences are noted in sections 2.8 and 5.

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
14. [Fast connection: SASL2, Bind 2, FAST (optional)](#14-fast-connection-sasl2-bind-2-fast-optional)

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

VPS <IP_VPS>, Fedora Server 44, SELinux enforcing, firewalld
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
- **Prosody 13 from the Fedora repository.** Fedora 44 has version 13.0.6. Fixes for the 13.x branch arrive through the `updates` repository together with the other system updates. No third-party repositories are needed.
- **Port 443 belongs to Prosody.** The `mod_net_multiplex` module accepts Direct TLS for clients (XEP-0368) and HTTPS for file sharing on the same port. The service is selected by ALPN, and if the client sent no ALPN, by the first bytes of the connection. Port 5222 stays for clients without XEP-0368 support.
- **SELinux in enforcing mode.** The Fedora policy has a module for Prosody: the service gets access only to its own files and ports. Port 443 is allowed with a standard boolean, no custom policy module is needed.
- **SQLite storage.** Accounts, contacts, the message archive and group chat data are stored in one file, `/var/lib/prosody/prosody.sqlite`. A copy of the database is taken on a running server with `sqlite3 .backup`, and moving to another server comes down to copying one file. The driver is part of the `lua-dbi` package (section 4.2). Uploaded files are stored on disk, not in the database.
- **Server-to-server connections over 5269 only.** Certificate verification for other servers is not configured on port 443, and with `s2s_secure_auth = true` servers fail authentication without it. So the `_xmpps-server` record is not published.
- **Calls through coturn.** `mod_turn_external` only hands clients the address and temporary credentials of the TURN server. The TURN server itself is installed separately (section 8).
- **Dedicated server.** Basic server hardening is part of this guide (section 2).

### 1.4. Limitations

- Port 443 is taken by Prosody. To host a website or a VPN on port 443 of this IP later, you will need an SNI router in front of the services (HAProxy, for example).
- Calls use port 3478 and the 50000-50100/udp range. On networks where only port 443 is open, calls may not work.
- Traffic is not hidden: the names `inkov.dev`, `upload.inkov.dev` and the XMPP protocol marker (ALPN `xmpp-client`) are sent in the unencrypted part of the TLS handshake.
- The domain in user addresses can only be changed together with the accounts: after a domain change they have to be created again.
- A new Fedora release comes out every six months, and each release is supported for about 13 months. Every 6-12 months the server has to be upgraded to the next release (section 11.6). If you want to upgrade the OS less often, use the Ubuntu 26.04 LTS guide.

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

- A VPS with a clean Fedora Server 44 and SSH access: root or a sudo user created by the hosting provider.
- An SSH key on your local computer. If you do not have one, it is created in section 2.3.
- Access to the Cloudflare dashboard for the `inkov.dev` domain.
- Access to the VPS console in the provider's control panel (VNC or similar) in case of a mistake in the SSH or firewall setup.
- An XMPP client on a phone or computer (section 10).

---

## 2. Preparing the server

### 2.1. OS and resources

```bash
# OS version: expect Fedora release 44
cat /etc/fedora-release
# Edition: Server Edition or Cloud Edition
grep '^VARIANT=' /etc/os-release
# SELinux mode: expect Enforcing
getenforce
# Free space on the root partition
df -h /
# Free memory
free -h
```

**Expected:** `Fedora release 44 (Forty Four)`, `VARIANT="Server Edition"` (`Cloud Edition` in cloud images), `Enforcing`, at least 6 GB of free disk space: up to 5 GB is reserved for user files (section 7). If you have less, lower `http_file_share_global_quota` in the Prosody configuration. Prosody and coturn together use less than 100 MB of memory.

If `getenforce` prints `Permissive` or `Disabled`, the provider has relaxed or disabled SELinux. The guide works in that case too, but without the protection that the SELinux policy gives. How to switch back to enforcing mode is described in the Fedora documentation (link in section 13).

### 2.2. Mirrors and system update

```bash
sudo dnf makecache
```

**Expected:** a list of repositories and the line `Metadata cache created.` with no `Curl error` or `Failed to download metadata` errors. In that case there is no need to change mirrors: dnf picks the nearest mirror itself through the `mirrors.fedoraproject.org` service. An error for the `fedora-cisco-openh264` repository alone (codecs for workstations) does not affect the server: dnf skips that repository.

If dnf cannot download metadata for the `fedora` and `updates` repositories, switch them to the Yandex mirror. The override goes into a separate file, and the `fedora-repos` package files in `/etc/yum.repos.d` stay unchanged.

```bash
sudo install -d -m 755 /etc/dnf/repos.override.d
sudo tee /etc/dnf/repos.override.d/80-mirror-yandex.repo > /dev/null <<'EOF'
# Yandex mirror instead of mirror selection through mirrors.fedoraproject.org
[fedora]
metalink=
baseurl=https://mirror.yandex.ru/fedora/linux/releases/$releasever/Everything/$basearch/os/

[updates]
metalink=
baseurl=https://mirror.yandex.ru/fedora/linux/updates/$releasever/Everything/$basearch/
EOF
sudo dnf clean all
sudo dnf makecache
# Addresses dnf uses now
sudo dnf repo info fedora updates | grep 'Base URL'
```

**Expected:**

```text
  Base URL           : https://mirror.yandex.ru/fedora/linux/releases/44/Everything/x86_64/os/
  Base URL           : https://mirror.yandex.ru/fedora/linux/updates/44/Everything/x86_64/
```

The `$releasever` variable is filled in automatically, so the file keeps working after the move to the next Fedora release. Rollback:

```bash
sudo rm /etc/dnf/repos.override.d/80-mirror-yandex.repo && sudo dnf makecache
```

Package upgrade:

```bash
sudo dnf upgrade --refresh -y
# Check whether kernel and system library updates require a reboot
sudo dnf needs-restarting
```

**Expected:** `Reboot should not be necessary.` - no reboot is needed. `Reboot is required to fully utilize these updates.` - run `sudo reboot` and reconnect. If the `needs-restarting` command is not found, install the dnf plugins: `sudo dnf install dnf5-plugins`.

### 2.3. Sudo user and key-based login

There is no need to work as root all the time: a separate user with sudo limits the consequences of mistakes. If the provider gave you root access only, create a user:

```bash
# On the VPS as root. On Fedora, sudo rights come from the wheel group
useradd -m -G wheel <USER>
# The password is needed for sudo
passwd <USER>
```

`useradd` asks no questions, and the password is set separately with `passwd`. If the provider has already created a user (in cloud images it is usually `fedora`), you can work as that user: `id` must show the `wheel` group.

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
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <USER>@<IP_VPS> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && restorecon -R ~/.ssh"
```

`restorecon` restores the SELinux labels on the `~/.ssh` directory. With a wrong label sshd cannot read `authorized_keys` and rejects the key.

If the provider has already set up root login with a key and disabled password login, copy root's key to the new user:

```bash
# On the VPS as root
install -d -m 700 -o <USER> -g <USER> /home/<USER>/.ssh
install -m 600 -o <USER> -g <USER> /root/.ssh/authorized_keys /home/<USER>/.ssh/authorized_keys
# SELinux labels for the key directory
restorecon -R /home/<USER>/.ssh
```

**Expected:** `ssh <USER>@<IP_VPS>` logs in without the account password (it may only ask for the key passphrase), `sudo -v` accepts the user's password. `sudo --version` shows `Sudo version 1.9.17p2`: Fedora uses the classic sudo implementation. From here on, all commands run as `<USER>` through `sudo`.

### 2.4. Key-only login

> **Important:** do this step only after the check in 2.3 has passed. Do not close the current SSH session until you have tested a new connection.

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null <<'EOF'
# SSH key login only, root login disabled
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
# Check syntax and apply. On Fedora the service is called sshd
sudo sshd -t && sudo systemctl restart sshd
# Effective values
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

**Expected:**

```text
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
```

sshd reads the files in `/etc/ssh/sshd_config.d/` in alphabetical order and uses the first value it finds for each parameter. On Fedora the directory already has `40-redhat-crypto-policies.conf` and `50-redhat.conf`, and cloud images sometimes add `50-cloud-init.conf` with `PasswordAuthentication yes`. The new file name starts with `10-`, so it is read first.

Check from your local computer in a new terminal window:

```bash
# Key login works
ssh <USER>@<IP_VPS> true && echo "key login works"
# Password login is rejected: expect "Permission denied (publickey)"
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password <USER>@<IP_VPS>
```

The message may also list `gssapi-keyex,gssapi-with-mic`: on Fedora GSSAPI is enabled in `50-redhat.conf`, and without a configured Kerberos setup it cannot be used to log in.

Rollback (if you lose access, use the provider's web console): `sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl restart sshd`.

### 2.5. Time

```bash
timedatectl
```

**Expected:** `System clock synchronized: yes` and `NTP service: active`. A wrong clock causes certificate verification errors. On Fedora time is synchronized by chrony. If synchronization is off:

```bash
sudo systemctl enable --now chronyd
# chrony status: expect Leap status: Normal
chronyc tracking
```

If the `chronyd` service is not found, install chrony: `sudo dnf install chrony`.

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

**Expected:** the two port checks print nothing. The first output shows SSH on port 22, the system DNS resolver `systemd-resolved` on `127.0.0.53:53` and `127.0.0.54:53`, and chronyd on `127.0.0.1:323`. On Fedora Server port 9090 is held by `systemd`: this is the Cockpit web console (section 2.8). `systemd-resolved` may also listen on port 5355 (LLMNR): the firewall closes it to the outside.

If the provider preinstalled a web server and port 443 is taken, stop it: for example, `sudo systemctl disable --now httpd` or `sudo systemctl disable --now nginx`.

### 2.8. Cockpit web console

Fedora Server enables the Cockpit web console on port 9090 by default, and the firewalld zone lets connections to it through. Cockpit accepts login with the user's password, so it can be used to get into the server even after password login is disabled for SSH (section 2.4).

```bash
# enabled - Cockpit starts when something connects to port 9090
systemctl is-enabled cockpit.socket
# Disable
sudo systemctl disable --now cockpit.socket
```

**Expected:** running `systemctl is-enabled cockpit.socket` again prints `disabled`. The message `No such file or directory` means Cockpit is not installed (cloud images), and this step is not needed. The firewalld rule for Cockpit is removed in section 5.

If you need Cockpit, start it only while you work with it (`sudo systemctl start cockpit.socket`) and connect through an SSH tunnel, without opening port 9090 in the firewall: `ssh -L 9090:127.0.0.1:9090 <USER>@<IP_VPS>` on the local computer, then `https://localhost:9090` in the browser.

### 2.9. Automatic security updates

On Fedora automatic updates are handled by the dnf5-automatic plugin on a systemd timer. With the default settings it only downloads updates, so installing security updates is turned on in `/etc/dnf/automatic.conf`. The package ships this file empty, and the defaults are in `/usr/share/dnf5/dnf5-plugins/automatic.conf`.

```bash
sudo dnf install dnf5-plugin-automatic
sudo tee /etc/dnf/automatic.conf > /dev/null <<'EOF'
# Install security updates only, automatically, without a reboot.
# Defaults: /usr/share/dnf5/dnf5-plugins/automatic.conf
[commands]
upgrade_type = security
apply_updates = yes
reboot = never
EOF
sudo systemctl enable --now dnf5-automatic.timer
systemctl list-timers dnf5-automatic.timer --no-pager
```

**Expected:** `dnf5-automatic.timer` is listed with the time of the next run. The timer fires daily at about 6:00 with a random delay of up to an hour. Kernel updates need a reboot, and with `reboot = never` there is none: once a month check whether a reboot is needed (section 11.2).

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
# dig utility (once)
sudo dnf install bind-utils
dig +short CAA inkov.dev
```

- Empty output - there are no restrictions, nothing else to do.
- If there are records, one of them must be `0 issue "letsencrypt.org"`. If there are records with the `issuewild` tag, you also need `0 issuewild "letsencrypt.org"`, otherwise the wildcard certificate will not be issued. Add the missing record in Cloudflare: Type `CAA`, Name `@`, choose the tag in the **Tag** field, value `letsencrypt.org`.

GitHub Pages also gets its certificates from Let's Encrypt, so existing CAA records usually allow `letsencrypt.org` already.

### 3.3. Verification

Queries sent straight to the Cloudflare DNS server show the records right after they are saved, with no caching by intermediate resolvers.

```bash
# Cloudflare DNS server for the zone
NS=$(dig +short NS inkov.dev | head -n 1); echo "$NS"
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

### 4.1. Prosody package in Fedora

```bash
# Package version and repository
dnf info prosody | grep -E '^(Name|Version|Release|Repository)'
```

**Expected:**

```text
Name           : prosody
Version        : 13.0.6
Release        : 1.fc44
Repository     : updates
```

The version number may be newer: Fedora updates the package within the 13.x branch. The line `Repository : fedora` means the updates have not been downloaded yet: go through section 2.2.

### 4.2. Installation

```bash
sudo dnf install prosody lua-dbi
```

Packages:

- `prosody` - the server. dnf installs the recommended packages `lua-unbound` (DNS resolver) and `lua-readline` (line editing in `prosodyctl shell`) with it: on Fedora recommended dependencies are installed by default.
- `lua-dbi` - the LuaDBI driver; Prosody works with the SQLite database through it. The SQLite, MySQL and PostgreSQL drivers come in one package, so dnf also installs the client libraries `mariadb-connector-c` and `libpq`. Prosody uses only SQLite.
- `luarocks` is not needed. It is only required to install third-party modules with `prosodyctl install`, and it pulls in the gcc compiler.

Check:

```bash
sudo prosodyctl about | grep -E '^Prosody|Lua version|LuaRocks|LuaDBI|luaunbound'
# The SQLite driver loads in Lua 5.4
lua -e 'require "DBI"; print("LuaDBI: ok")'
# The service is not enabled after installation
systemctl is-enabled prosody
```

**Expected:**

```text
Prosody 13.0.6
Lua version:             	Lua 5.4
LuaRocks:        	Not installed
LuaDBI:       	0.7
luaunbound:   	1.1.0
LuaDBI: ok
disabled
```

If there is no `luaunbound` line, install the packages by hand: `sudo dnf install lua-unbound lua-readline`. The error `module 'DBI' not found` means `lua-dbi` is not installed. The Prosody service is not running yet; it is enabled in section 9.1, after configuration. The package has also created a self-signed certificate for `localhost` in `/etc/pki/prosody`, which this setup does not use.

### 4.3. Permission for port 443: systemd and SELinux

Prosody cannot listen on port 443 without two changes:

1. The `prosody` service from the package runs as the `prosody` user without the right to listen on ports below 1024. Without it the log shows a bind error on port 443 with `Permission denied`. The right is added with the `AmbientCapabilities` parameter in a drop-in file: the package file `/usr/lib/systemd/system/prosody.service` stays unchanged and is not overwritten on updates.
2. The SELinux policy allows Prosody the XMPP ports (5222, 5269) and 5280-5281. Port 443 has the type `http_port_t`, and the policy allows it only when the `prosody_bind_http_port` boolean is on. The boolean is off by default.

```bash
# Drop-in for the prosody service
sudo install -d -m 755 /etc/systemd/system/prosody.service.d
sudo tee /etc/systemd/system/prosody.service.d/10-bind-443.conf > /dev/null <<'EOF'
# Right to listen on ports below 1024 (443) without running as root (section 4.3)
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
EOF
sudo systemctl daemon-reload
# Effective value
systemctl show prosody -p AmbientCapabilities

# SELinux: Prosody may listen on ports of type http_port_t. -P keeps the value after a reboot
sudo setsebool -P prosody_bind_http_port on
getsebool prosody_bind_http_port
```

**Expected:**

```text
AmbientCapabilities=cap_net_bind_service
prosody_bind_http_port --> on
```

`setsebool -P` rebuilds the policy and can take up to a minute. Other ports also have the `http_port_t` type (80, 8443 and a few more), but Prosody listens only on the ports from its configuration.

Rollback:

```bash
sudo rm -r /etc/systemd/system/prosody.service.d && sudo systemctl daemon-reload
sudo setsebool -P prosody_bind_http_port off
```

---

## 5. Firewall

On Fedora Server firewalld is enabled after the system is installed. The rules are added to the default zone.

> **Important:** do not close the current SSH session until you have tested a new connection after changing the rules.

```bash
# firewalld state: expect running
sudo firewall-cmd --state
# Default zone and its rules
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --list-all
```

**Expected:** `running`, zone `FedoraServer` (`public` in cloud images and after a manual firewalld install). The `services:` line has `ssh`, and on Fedora Server also `cockpit` and `dhcpv6-client`.

If `firewall-cmd` is not found or reports `not running`, install and start firewalld. The default `public` zone allows SSH on port 22, so the current session will not be dropped.

```bash
sudo dnf install firewalld
# Only if SSH runs on a port other than 22: allow that port before starting firewalld
# sudo firewall-offline-cmd --add-port=<PORT>/tcp
sudo systemctl enable --now firewalld
```

Rules for Prosody:

```bash
# XMPP: Direct TLS and HTTPS (443), STARTTLS (5222), server-to-server connections (5269)
sudo firewall-cmd --permanent --add-service=https --add-service=xmpp-client --add-service=xmpp-server
# The Cockpit web console is not needed from outside (section 2.8)
sudo firewall-cmd --permanent --remove-service=cockpit
# Apply the permanent rules
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

**Expected:** each command prints `success`, and the last one prints the list of services:

```text
dhcpv6-client https ssh xmpp-client xmpp-server
```

In the `public` zone the list also has `mdns`. If Cockpit was not in the zone, the removal prints `Warning: NOT_ENABLED: cockpit`, which is expected. Then open a new SSH connection. If you lose access, run `sudo systemctl stop firewalld` in the provider's web console.

- firewalld services are named sets of ports: `https` is 443/tcp, `xmpp-client` is 5222/tcp, `xmpp-server` is 5269/tcp. The definitions are in `/usr/lib/firewalld/services/`.
- `--permanent` writes the rule to the configuration, and `--reload` applies it. A rule added without `--permanent` lasts only until firewalld is reloaded or restarted.
- Port 5280 is not opened: it is only needed locally. Port 80 is not needed: the certificate is issued through DNS.
- The coturn ports (3478 and 50000-50100/udp) are opened in section 8.3, after coturn is configured. Ports 5000 (Proxy65) and 5349 (TURN over TLS) are opened only if you enable those features (sections 7.3 and 8.4).
- The rules apply to both IPv4 and IPv6.
- If an external firewall is enabled in the provider's control panel, open the same ports there.
- To remove a rule, run the same command with `--remove-service` or `--remove-port`, then `sudo firewall-cmd --reload`.

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

The token is pasted in the editor and does not end up in the shell history. If nano is not installed: `sudo dnf install nano`.

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
sudo dnf install certbot python3-certbot-dns-cloudflare
certbot --version
```

**Expected:** `certbot 5.8.0` or newer. This is the version from the Fedora 44 repository; it supports DNS-01 through the Cloudflare API with a token. Do not install the snap version of certbot alongside it: both installations use the `/etc/letsencrypt` directory and run renewals independently of each other.

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

`--deploy-hook` runs after every successful issuance and renewal. `prosodyctl --root cert import` copies the certificate and key to the Prosody certificate directory (`/etc/pki/prosody`) with the owner `prosody` and reloads Prosody. You cannot point Prosody directly at the files in `/etc/letsencrypt`: the key there is readable only by root, and Prosody runs as the `prosody` user.

**Expected:** the output contains the lines

```text
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/inkov.dev/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/inkov.dev/privkey.pem
```

On the first issuance certbot also reports that the import command failed (`reported error code 1`), and the command output has the lines `No certificate for host localhost found :(` and `No certificates imported :(`. This is expected: the default Prosody configuration has only the `localhost` host, and the certificate has no name for it. The certificate is saved anyway, and the import command is stored in the renewal settings. The import is done in section 9.2, after Prosody is configured.

Check:

```bash
sudo certbot certificates
```

**Expected:** `Domains: inkov.dev *.inkov.dev`, `Expiry Date` about 90 days ahead.

### 6.7. Automatic renewal

certbot from the Fedora package installs the `certbot-renew.timer` timer but does not enable it. The timer runs `certbot renew` twice a day with a random delay. certbot renews the certificate in advance: for a 90-day certificate, about 30 days before it expires. `--deploy-hook` runs after the renewal.

```bash
# Enable the renewal timer
sudo systemctl enable --now certbot-renew.timer
systemctl list-timers certbot-renew.timer --no-pager
# The import command is saved in the renewal settings
sudo grep renew_hook /etc/letsencrypt/renewal/inkov.dev.conf
# Test renewal against the staging server
sudo certbot renew --dry-run
```

**Expected:** `certbot-renew.timer` is listed with the time of the next run; the line `renew_hook = prosodyctl --root cert import /etc/letsencrypt/live`; the message `Congratulations, all simulated renewals succeeded`.

The `certbot-renew.service` unit reads extra parameters from `/etc/sysconfig/certbot`. Leave the `PRE_HOOK`, `POST_HOOK` and `DEPLOY_HOOK` variables in this file empty: a `DEPLOY_HOOK` value replaces the import command saved for the certificate.

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
dig +short @"$(dig +short NS inkov.dev | head -n 1)" TXT _acme-challenge.inkov.dev
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

How the configuration is laid out in the Fedora package:

- The main file `/etc/prosody/prosody.cfg.lua` includes the `conf.d/*.cfg.lua` files at the end with an `Include` line. `conf.d` contains the examples `localhost.cfg.lua` and `example.com.cfg.lua`.
- `/etc/prosody/certs` is a symbolic link to `/etc/pki/prosody`, where the certificates are stored.

The new configuration is kept entirely in the main file and does not include `conf.d`, so the examples from the package are not used. There is no need to delete them.

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
| `pidfile` | `/run/prosody/prosody.pid` | `prosodyctl` uses it to reload the server, including after a certificate import. The path is from the Fedora package |
| `authentication` | `internal_hashed` | passwords are stored as SCRAM hashes |
| `storage`, `sql` | `sql`, driver `SQLite3`, file `prosody.sqlite` | all Prosody data in one database, `/var/lib/prosody/prosody.sqlite`. Prosody creates the tables itself on first start. Uploaded files are stored on disk |
| `archive_expires_after` | `1y` | history is kept on the server for a year and synced across devices. A shorter period means less data on the server. Example values: `1w`, `30d`, `6 months`, `1y`, `never`. The value `1m` is ambiguous (month or minute), and Prosody 13 logs it as an error |
| `turn_external_*` | `turn.inkov.dev`, secret, TCP | clients get the coturn address and temporary credentials valid for 24 hours |
| `certificates` | `/etc/pki/prosody` | certificate directory from the Fedora package. Prosody finds the certificate for each host in it, and for subdomains uses the wildcard |

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
- `proxy.inkov.dev` (`proxy65`) - direct file transfer for old clients. Modern clients use `http_file_share`, so the component is commented out. To enable it, remove `--` from the two lines, create the `proxy` DNS record (section 3.1) and open the port: `sudo firewall-cmd --permanent --add-port=5000/tcp && sudo firewall-cmd --reload`. The SELinux policy allows Prosody to use port 5000 with no extra settings.

### 7.4. Configuration file

Copy the whole block and run it in the terminal: the command replaces the contents of the file.

```bash
sudo tee /etc/prosody/prosody.cfg.lua > /dev/null <<'EOF'
-- /etc/prosody/prosody.cfg.lua
-- Personal XMPP server for addresses like user@inkov.dev.
-- Based on the configuration from the Prosody 13.0 package in Fedora 44.
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
pidfile = "/run/prosody/prosody.pid"

-- Passwords are stored as hashes (SCRAM)
authentication = "internal_hashed"

-- Storage: an SQLite database in a single file. Requires the driver from the lua-dbi package (section 4.2).
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

-- TURN server for calls (coturn). The secret matches static-auth-secret in /etc/coturn/turnserver.conf.
-- If you do not need calls, delete these three lines and "turn_external" from modules_enabled.
turn_external_host = "turn.inkov.dev"
turn_external_secret = "<TURN_SECRET>"
turn_external_tcp = true

-- Logs
log = {
    info = "/var/log/prosody/prosody.log"; -- change info to debug for a detailed log
    error = "/var/log/prosody/prosody.err";
}

-- Certificate directory. On Fedora /etc/prosody/certs is a link to it.
certificates = "/etc/pki/prosody"

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
# Owner root, group prosody, no access for others (same as in the package)
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

The message about `check features` refers to browser connections (section 9.3) and does not affect the server. The new configuration takes effect when Prosody starts in section 9.1.

---

## 8. Calls: coturn

If you do not need calls, skip this section. In that case delete the `"turn_external";` line and the three `turn_external_*` lines from `/etc/prosody/prosody.cfg.lua`.

### 8.1. Installation

```bash
sudo dnf install coturn
# The service is not running after installation: expect inactive
systemctl is-active coturn
```

> **Important:** with the configuration file from the package, coturn allocates relay ports without checking credentials. So the service is started only after it is configured, and the TURN ports are opened in the firewall only once it runs with the new configuration (section 8.3).

### 8.2. Configuration

```bash
# Backup of the file from the package
sudo cp /etc/coturn/turnserver.conf /etc/coturn/turnserver.conf.orig
sudo tee /etc/coturn/turnserver.conf > /dev/null <<'EOF'
# /etc/coturn/turnserver.conf - TURN server for XMPP calls

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

# TLS is not used (see section 8.4). DTLS is off by default in coturn 4.18.
no-tls

# Security. The management console (CLI) and the SOFTWARE attribute with the coturn version
# are off by default in coturn 4.18.
fingerprint
no-multicast-peers

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

# Log to a file, as in the package configuration. Rotation is done by logrotate.
# With syslog instead of a file, coturn 4.18 exits on the HUP signal
# that systemctl reload coturn sends.
log-file=/var/log/coturn/turnserver.log
simple-log
EOF
# TURN secret from the file in place of <TURN_SECRET>
sudo sh -c 'sed -i "s/<TURN_SECRET>/$(cat /root/.secrets/turn_secret)/" /etc/coturn/turnserver.conf'
# Only root and the coturn group can read the file (same as in the package)
sudo chown root:coturn /etc/coturn/turnserver.conf
sudo chmod 640 /etc/coturn/turnserver.conf
```

If section 2.6 showed that the VPS is behind NAT, uncomment the `external-ip` line and set the public and internal addresses: `sudo nano /etc/coturn/turnserver.conf`.

What the parameters do:

- `use-auth-secret` and `static-auth-secret` - Prosody gives clients a temporary username and password derived from the shared secret. Permanent TURN passwords are not needed.
- `min-port` and `max-port` - 101 ports for media. One call through TURN takes several ports, which is enough for a personal server.
- `denied-peer-ip` - blocks relaying to internal addresses. Without it a TURN user can reach services that listen only on `127.0.0.1` or on the provider's internal network.
- `log-file` and `simple-log` - the log goes to `/var/log/coturn/turnserver.log`, as in the package configuration. The package sets up its rotation with `systemctl try-reload-or-restart coturn`.
- In coturn 4.18 the management console, DTLS and the SOFTWARE attribute with the version are off by default, and the minimum TLS version is 1.2. The `no-cli`, `no-dtls`, `no-software-attribute`, `no-tlsv1` and `no-tlsv1_1` parameters from configurations for older versions are not needed: for `no-dtls` coturn 4.18 prints `Bad configuration format`, and for `no-cli` it prints a message that the option is deprecated.

SELinux does not affect coturn: the Fedora policy has no separate module for coturn.

### 8.3. Startup and verification

```bash
# Start now and on boot
sudo systemctl enable --now coturn
systemctl status coturn --no-pager
sudo ss -tulpn | grep turnserver
# Warnings at startup
sudo grep -E 'WARNING|ERROR' /var/log/coturn/turnserver.log
```

**Expected:** `Active: active (running)`, `turnserver` listens on port 3478 over UDP and TCP. Only these warnings are expected in the log (each line starts with the date and time):

```text
WARNING Certificate file not found or not readable: //turn_server_cert.pem
WARNING Private key file not found or not readable: //turn_server_pkey.pem
WARNING NO EXPLICIT LISTENER ADDRESS(ES) ARE CONFIGURED
WARNING NO EXPLICIT RELAY ADDRESS(ES) ARE CONFIGURED
```

The certificate is not needed while TLS is off. Without explicit addresses coturn listens on all addresses of the server.

TURN ports in the firewall: the `stun` service (3478/tcp and 3478/udp) and the media relay range.

```bash
sudo firewall-cmd --permanent --add-service=stun --add-port=50000-50100/udp
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --list-ports
```

**Expected:** the list of services has `stun`, and the list of ports has `50000-50100/udp`. If the provider has an external firewall, open the same ports there.

Check that Prosody and coturn work together. The command reads the `turn_external_*` settings from the Prosody configuration, so Prosody does not have to be running. The name `turn.inkov.dev` must already resolve (section 3).

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

The coturn package creates directories for certificates: `/etc/pki/coturn/public` (readable by everyone) and `/etc/pki/coturn/private` (root and the `coturn` group only). The script below copies the certificate into them. certbot runs the scripts in `/etc/letsencrypt/renewal-hooks/deploy` after every renewal.

```bash
sudo install -d /etc/letsencrypt/renewal-hooks/deploy
sudo tee /etc/letsencrypt/renewal-hooks/deploy/coturn.sh > /dev/null <<'EOF'
#!/bin/sh
# Copies the Let's Encrypt certificate for coturn. coturn rereads the certificate
# on systemctl reload, calls in progress are not dropped.
set -e
install -o root -g root -m 644 /etc/letsencrypt/live/inkov.dev/fullchain.pem /etc/pki/coturn/public/fullchain.pem
install -o root -g coturn -m 640 /etc/letsencrypt/live/inkov.dev/privkey.pem /etc/pki/coturn/private/privkey.pem
systemctl try-reload-or-restart coturn
EOF
sudo chmod 755 /etc/letsencrypt/renewal-hooks/deploy/coturn.sh
```

Enabling TLS in coturn and Prosody:

```bash
# First certificate copy
sudo /etc/letsencrypt/renewal-hooks/deploy/coturn.sh
# coturn: remove the TLS restriction, add the port and certificate
sudo sed -i '/^no-tls$/d' /etc/coturn/turnserver.conf
sudo tee -a /etc/coturn/turnserver.conf > /dev/null <<'EOF'

# TLS (section 8.4). The minimum TLS version in coturn 4.18 is 1.2.
tls-listening-port=5349
cert=/etc/pki/coturn/public/fullchain.pem
pkey=/etc/pki/coturn/private/privkey.pem
EOF
# coturn opens a new port only on restart
sudo systemctl restart coturn
# Prosody: advertise the TLS TURN port to clients
sudo sed -i 's/^turn_external_tcp = true$/&\nturn_external_tls_port = 5349/' /etc/prosody/prosody.cfg.lua
# Firewall: TCP only, DTLS is not used
sudo firewall-cmd --permanent --add-port=5349/tcp
sudo firewall-cmd --reload
```

If Prosody is already running (section 9.1), run `sudo systemctl restart prosody`: `mod_turn_external` reads its parameters when the module loads, and `reload` will not pick up the new port.

Check:

```bash
openssl s_client -connect turn.inkov.dev:5349 </dev/null 2>/dev/null | openssl x509 -noout -enddate -ext subjectAltName
```

**Expected:** a line with the expiry date and a line with `DNS:inkov.dev, DNS:*.inkov.dev`. When the certificate is renewed, coturn rereads it on a signal, and calls in progress are not dropped.

---

## 9. Startup and verification

### 9.1. Starting Prosody

```bash
# Start now and on boot
sudo systemctl enable --now prosody
systemctl status prosody --no-pager
# Latest log entries and the error file
sudo tail -n 30 /var/log/prosody/prosody.log
sudo cat /var/log/prosody/prosody.err
```

**Expected:** `Active: active (running)`. The log contains these lines:

```text
portmanager	info	Activated service 's2s' on [::]:5269, [*]:5269
portmanager	info	Activated service 'multiplex_ssl' on [::]:443, [*]:443
portmanager	info	Activated service 'c2s' on [::]:5222, [*]:5222
portmanager	info	Activated service 'http' on [127.0.0.1]:5280
portmanager	info	Activated service 'https' on no ports
upload.inkov.dev:http	info	Serving 'file_share' at https://upload.inkov.dev/file_share
```

Port 443 is served by `multiplex_ssl`, and the line `'https' on no ports` is expected: HTTPS runs on 443. `prosody.err` is empty or does not exist. If the service did not start, `journalctl -u prosody -n 50 --no-pager` shows the reason.

If the log shows a bind error on port 443 with `Permission denied`, check both changes from section 4.3 and look for SELinux denials:

```bash
systemctl show prosody -p AmbientCapabilities
getsebool prosody_bind_http_port
# SELinux denials in the last 10 minutes: expect <no matches>
sudo ausearch -m avc -ts recent
```

Check the SQLite database:

```bash
sudo ls -l /var/lib/prosody/prosody.sqlite
sudo grep -c -E 'LuaDBI or LuaSQLite3|no data storage' /var/log/prosody/prosody.err
```

**Expected:** the `prosody.sqlite` file owned by `prosody` with permissions `-rw-r-----`, error count `0`. If the file is missing or the count is above zero, Prosody is running without storage: check the driver (section 4.2).

### 9.2. Certificate import and renewal test

certbot runs this same command after every renewal. The configuration now has the `inkov.dev` host and the components, so the import succeeds:

```bash
sudo prosodyctl --root cert import /etc/letsencrypt/live
sudo ls -l /etc/pki/prosody/
```

**Expected:** `Imported certificate and key for hosts inkov.dev, upload.inkov.dev, conference.inkov.dev`. `/etc/pki/prosody` now contains `inkov.dev.crt` and `inkov.dev.key` owned by `prosody`. Prosody reloads the certificates automatically, and `Certificates reloaded` lines appear in the log.

Renewal test together with the import command:

```bash
sudo certbot renew --dry-run --run-deploy-hooks
```

**Expected:** `Congratulations, all simulated renewals succeeded` and the import command message `Imported certificate and key for hosts ...`. If TLS for coturn is enabled (8.4), the `coturn.sh` script runs as well.

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
| `check certs` | certificates for `inkov.dev` and the components, names and validity | `Certificate: /etc/pki/prosody/inkov.dev.crt` for each host |
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
- the lines `<mechanism>SCRAM-SHA-1</mechanism>`, `<mechanism>SCRAM-SHA-1-PLUS</mechanism>`, `<mechanism>PLAIN</mechanism>` and `<mechanism>OAUTHBEARER</mechanism>`;
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

- Login to the server is by SSH key only, root login is disabled (section 2.4). The Cockpit web console is disabled and closed in the firewall (sections 2.8 and 5).
- SELinux runs in enforcing mode. Do not disable it to get rid of errors: `sudo ausearch -m avc -ts recent` shows why access was denied.
- Registration is closed (`allow_registration = false`), only the administrator creates accounts.
- Use different long passwords for XMPP and sudo.
- Port 5280 and the `prosodyctl shell` console are available only locally, do not expose them.
- The Cloudflare token can change DNS in the `inkov.dev` zone. If you suspect a leak, delete it in **My Profile → API Tokens**, create a new one and update `/root/.secrets/cloudflare.ini`.

### 11.2. Updates

- Security updates are installed by dnf5-automatic (section 2.9).
- Once a month install the remaining updates and check whether a reboot is needed:

```bash
sudo dnf upgrade --refresh
sudo dnf needs-restarting
```

- Prosody is updated from the Fedora repository together with the rest of the system. Within one Fedora release it stays on the 13.x branch. The major Prosody version can change when you move to the next Fedora release (section 11.6). To update Prosody only by hand: `sudo dnf versionlock add prosody`; to remove the lock: `sudo dnf versionlock delete prosody`.
- When the `prosody` and `coturn` packages are updated, dnf restarts their services at the end of the transaction, including during the automatic installation of security updates. Clients reconnect on their own, and calls going through TURN at that moment are dropped. `sudo dnf needs-restarting -s` lists the services that still run with old library versions.

### 11.3. Backups

Do not copy the database file with `cp` or `tar` while Prosody is running: the copy may be inconsistent. The `sqlite3` utility takes a consistent copy without stopping the server.

```bash
# sqlite3 utility (once)
sudo dnf install sqlite
# Consistent database copy on a running server
sudo sqlite3 /var/lib/prosody/prosody.sqlite ".backup /root/prosody-$(date +%F).sqlite"
# Check the copy: expect ok
sudo sqlite3 /root/prosody-$(date +%F).sqlite "PRAGMA integrity_check;"
# Archive: database copy, configuration, certificates, secrets and the service drop-in.
# Files uploaded by users are not included. --ignore-failed-read skips
# missing paths (coturn and the modules from section 14 are optional)
sudo tar --ignore-failed-read -czf /root/xmpp-backup-$(date +%F).tar.gz \
  /root/prosody-$(date +%F).sqlite /etc/prosody /etc/pki/prosody /etc/letsencrypt \
  /etc/systemd/system/prosody.service.d /etc/coturn /etc/pki/coturn \
  /usr/local/lib/prosody /root/.secrets
sudo ls -lh /root/xmpp-backup-*.tar.gz
```

The tar messages `Removing leading ... from member names` and `Warning: Cannot stat: No such file or directory` for optional paths that do not exist on the server are expected. Copying the archive to your local computer:

```bash
# On the VPS: copy the archive to your home directory, readable only by you
sudo install -m 600 -o "$USER" /root/xmpp-backup-$(date +%F).tar.gz ~/
# On the local computer
scp <USER>@xmpp.inkov.dev:~/xmpp-backup-*.tar.gz .
# On the VPS: delete the copy from your home directory
rm ~/xmpp-backup-*.tar.gz
```

Restore on a server prepared according to sections 2-8:

```bash
# Configuration files, certificates, secrets and the database copy in /root
sudo tar -xzf xmpp-backup-<date>.tar.gz -C /
# SELinux labels for the restored files
sudo restorecon -R /etc/prosody /etc/pki/prosody /etc/letsencrypt /etc/coturn /etc/pki/coturn /etc/systemd/system/prosody.service.d /root/.secrets
[ -d /usr/local/lib/prosody ] && sudo restorecon -R /usr/local/lib/prosody
sudo systemctl daemon-reload
# The database is replaced while Prosody is stopped
sudo systemctl stop prosody
sudo install -o prosody -g prosody -m 640 /root/prosody-<date>.sqlite /var/lib/prosody/prosody.sqlite
sudo restorecon /var/lib/prosody/prosody.sqlite
sudo systemctl start prosody
sudo systemctl restart coturn
sudo prosodyctl shell user list inkov.dev
```

The SELinux boolean is not part of the archive: on the new server run `sudo setsebool -P prosody_bind_http_port on` (section 4.3).

> **Important:** the archive contains the database with accounts and message history, the certificate private key, the Cloudflare token and the TURN secret. Store the copy encrypted.

### 11.4. Logs

- Prosody: `/var/log/prosody/prosody.log` and `prosody.err`. The package sets up rotation with logrotate.
- coturn: `/var/log/coturn/turnserver.log`.
- certbot: `/var/log/letsencrypt/letsencrypt.log`.
- SSH: `journalctl -u sshd`.
- SELinux: `sudo ausearch -m avc -ts today`. If the command is not found: `sudo dnf install audit`.

SELinux denials with `name_connect` for `prosody_t` to ports of other servers can show up, and this is expected. Prosody 13 first tries to connect to another server over Direct TLS if that server has an `_xmpps-server` record. The policy allows Prosody outgoing connections only to XMPP ports, so after the denial Prosody connects over 5269.

### 11.5. fail2ban

Optional. SSH login is possible only with a key, so SSH password guessing is not possible. By default Prosody 13 does not log the IP addresses of failed login attempts, so fail2ban needs the third-party module `mod_log_auth` (https://modules.prosody.im/mod_log_auth). With long random passwords brute force is not effective, so fail2ban is optional for a personal server.

### 11.6. Support period and upgrading to the next Fedora release

Fedora 44 was released on April 28, 2026. Support ends 4 weeks after the Fedora 46 release, which is June 2, 2027 according to the current schedule. After that date no security updates are published. Move to the next release within a few months after it comes out, without waiting for the end of support.

1. Make a backup (11.3).
2. Check which Prosody version the new release has. If the major version changes (for example, from 13 to 14), read the release notes at https://prosody.im. If you have set up section 14, check that the modules are compatible (14.8).
3. Run the upgrade:

```bash
# Install all updates for the current release
sudo dnf upgrade --refresh
# Versions in the next release (45 is the number of the new release)
dnf repoquery --releasever=45 --latest-limit 1 prosody coturn certbot
# Download the packages of the new release
sudo dnf system-upgrade download --releasever=45
# Reboot and install. The server is unavailable for a few minutes
sudo dnf system-upgrade reboot
```

4. Check the system and the services:

```bash
cat /etc/fedora-release
getsebool prosody_bind_http_port
sudo prosodyctl check
systemctl status prosody coturn --no-pager
sudo certbot renew --dry-run
```

The SELinux boolean, the service drop-in, the firewalld rules and the mirror override from section 2.2 are kept during the upgrade. The upgrade procedure is described in the Fedora documentation (link in section 13).

---

## 12. Troubleshooting

| Symptom | Likely cause | How to check | How to fix |
|---|---|---|---|
| The Prosody log shows a bind error on port 443 with `Permission denied` | No drop-in with `AmbientCapabilities`, or `prosody_bind_http_port` is off | `systemctl show prosody -p AmbientCapabilities`, `getsebool prosody_bind_http_port`, `sudo ausearch -m avc -ts recent` | Repeat section 4.3, then `sudo systemctl restart prosody` |
| Prosody was started by hand (`prosody` or `prosodyctl start`), port 443 does not open | Only the systemd service gets the right to use the port | `systemctl status prosody --no-pager` | Stop the manually started process and use `sudo systemctl start prosody` |
| `prosody.err` contains `LuaDBI or LuaSQLite3 are required for using SQL databases`, `prosodyctl adduser` reports `no data storage active`, login does not work | `lua-dbi` is not installed | `lua -e 'require "DBI"'` | `sudo dnf install lua-dbi`, then `sudo systemctl restart prosody` |
| `prosody.err` grows fast and contains long `stack traceback` entries | Most often storage is not working (see the row above) | `sudo grep -m 5 -v -E '^\s' /var/log/prosody/prosody.err` | Fix the first error that comes before the traceback |
| Prosody does not start after a configuration change | Lua syntax error: a missing quote, bracket or `;` | `sudo prosodyctl check config` shows the line with the error | Fix the line or restore `/etc/prosody/prosody.cfg.lua.orig` |
| The log shows `Address already in use` | The port is taken by another process | `sudo ss -tulpn`, find the port in the output | Stop that process (for example, a preinstalled web server on 443) |
| `dnf makecache`: `Curl error` or `Failed to download metadata` for `fedora` and `updates` | The mirror selection service or the mirrors are unreachable from the provider's network | `curl -sI 'https://mirrors.fedoraproject.org/metalink?repo=fedora-44&arch=x86_64'` | Yandex mirror (section 2.2) |
| On the first issuance certbot reports `reported error code 1` and `No certificates imported :(` | The default Prosody configuration has no `inkov.dev` host | - | Nothing to do: the import is done in section 9.2 |
| The certificate is not renewed, `systemctl list-timers` does not show `certbot-renew.timer` | The timer was not enabled after certbot was installed | `systemctl is-enabled certbot-renew.timer` | `sudo systemctl enable --now certbot-renew.timer` |
| After renewal Prosody still has the old certificate | `/etc/sysconfig/certbot` sets `DEPLOY_HOOK`, which replaces the import command | `grep HOOK /etc/sysconfig/certbot` | Clear `DEPLOY_HOOK`, run `sudo prosodyctl --root cert import /etc/letsencrypt/live` |
| The client cannot find the server | SRV records are not created or not yet visible to the resolver; ports 443 and 5222 are closed | `dig +short SRV _xmpps-client._tcp.inkov.dev`, `nc -vz xmpp.inkov.dev 443` | Check the records (section 3), firewalld (`sudo firewall-cmd --list-services`) and the provider's firewall |
| The client connects over 5222 but not over 443 | No `_xmpps-client` record, the client does not support XEP-0368, or port 443 is closed | Checks from 9.4 | Create the record, open the port. Clients without XEP-0368 connect to 5222 |
| The client reports a certificate error | The certificate does not include `inkov.dev`, was not imported, or has expired | The `openssl` commands from 9.4, `sudo prosodyctl check certs` | Issue the certificate with `-d inkov.dev -d '*.inkov.dev'`, run `sudo prosodyctl --root cert import /etc/letsencrypt/live` |
| certbot: `Error determining zone_id: 9109 ... Did you enter a valid Cloudflare Token?` | Wrong token | The token check from 6.3 | Copy the token again or create a new one |
| certbot: `Unable to determine zone_id for inkov.dev` | The token has no access to the zone | Token settings in Cloudflare | Zone Resources: `Include` / `Specific zone` / `inkov.dev` |
| certbot: `Error communicating with the Cloudflare API` with a hint about `Zone:DNS:Edit` | The token has no permission to edit DNS | Token permissions | `Zone` / `DNS` / `Edit` |
| certbot or the token check times out | The Cloudflare API is unreachable from the provider's network | The token check from 6.3 | Manual DNS-01 challenge (6.8, option 1) |
| certbot: `DNS problem: NXDOMAIN looking up TXT` or `Incorrect TXT record` | The TXT record did not propagate in time | Run the command again | Increase `--dns-cloudflare-propagation-seconds` to 120 |
| Messages to other servers are not delivered | Port 5269 is closed, no `_xmpp-server` SRV record, the remote server has an invalid certificate | `nc -vz xmpp.inkov.dev 5269`, `sudo tail -n 50 /var/log/prosody/prosody.log` | Open the port, check the SRV record; for a single domain with an invalid certificate, add `s2s_insecure_domains = { "example.org" }` to the global part of the configuration |
| The audit log shows `name_connect` denials for `prosody_t` | An outgoing Direct TLS connection attempt to a server with an `_xmpps-server` record | `sudo ausearch -m avc -ts today` | Nothing to do: Prosody connects to these servers over 5269 (section 11.4) |
| Files are not sent | Port 443 is closed, no `upload` record, a limit is exceeded | `nc -vz xmpp.inkov.dev 443`, `curl` from 9.4 | Open the port, create the record, change the limits in the `upload.inkov.dev` component |
| File links contain `:5280` or `http://` | The `upload.inkov.dev` component has no `http_external_url` | The `Serving 'file_share' at` line in the Prosody log | Add `http_external_url = "https://upload.inkov.dev/"` to the component and restart Prosody |
| Calls do not connect or there is no audio | coturn is stopped, the 50000-50100/udp range is closed, the secrets do not match, the VPS is behind NAT without `external-ip` | `systemctl status coturn`, `sudo prosodyctl check turn -v --ping=stun.conversations.im` | Start coturn, open the ports, insert the secret again, set `external-ip` |
| `check turn`: `STUN returned a private IP` | The VPS is behind NAT | `ip -br addr` | `external-ip=<IP_VPS>/<INTERNAL_IP>` in `/etc/coturn/turnserver.conf`, then `sudo systemctl restart coturn` |
| coturn stops after `systemctl reload coturn` or log rotation | The configuration has `syslog` instead of `log-file`: coturn 4.18 exits on the HUP signal | `grep -e '^syslog' -e '^log-file' /etc/coturn/turnserver.conf` | Put back `log-file` and `simple-log` (section 8.2), `sudo systemctl restart coturn` |
| The coturn log shows `Bad configuration format: no-dtls` or `no-cli option is deprecated` | Parameters from a configuration for older coturn versions | `sudo grep -e WARNING -e ERROR /var/log/coturn/turnserver.log` | Delete `no-dtls`, `no-cli`, `no-software-attribute`, `no-tlsv1`, `no-tlsv1_1` |
| Prosody: `Permission denied` when loading the key | The configuration has a path to the key in `/etc/letsencrypt` | `sudo cat /var/log/prosody/prosody.err` | Remove the explicit certificate paths, run `cert import` |
| SSH login fails after section 2.4 | The key was not copied, login was tested as a different user, or `~/.ssh` has a wrong SELinux label | Provider's web console: `ls -lZ /home/<USER>/.ssh/authorized_keys` | `sudo restorecon -R /home/<USER>/.ssh`; if that does not help, `sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl restart sshd` and repeat section 2.3 |
| SSH access is lost after changing the firewalld rules | The `ssh` service was removed, or SSH runs on a different port | Provider's web console: `sudo firewall-cmd --list-all` | `sudo firewall-cmd --permanent --add-service=ssh` (or `--add-port=<PORT>/tcp`), `sudo firewall-cmd --reload` |
| `check dns`: `No _xmpps-server SRV record found ..., but it looks like you need one.` | A quirk of the check when `net_multiplex` is used | Section 9.3 | Nothing to do: server-to-server connections go over 5269 |
| `check dns`: `inkov.dev A record points to unknown address 185.199.x.x` | The server's resolver cannot see the SRV records | `dig +short SRV _xmpp-client._tcp.inkov.dev` on the VPS | Check the SRV records in Cloudflare, wait up to 30 minutes |

Detailed Prosody log: in `/etc/prosody/prosody.cfg.lua` change `info =` to `debug =` in the `log` block, run `sudo systemctl restart prosody`, reproduce the problem and switch back to `info`.

---

## 13. Summary

1. Prosody 13 from the Fedora repository on a Fedora Server 44 VPS serves `user@inkov.dev` addresses, while the `inkov.dev` site stays on GitHub Pages. Data is stored in the SQLite database `/var/lib/prosody/prosody.sqlite`.
2. Clients connect over port 443 (Direct TLS) or 5222, other servers over 5269. SRV records in the `inkov.dev` zone point to `xmpp.inkov.dev`.
3. Port 443 is opened to Prosody through `AmbientCapabilities` in a systemd drop-in and the SELinux boolean `prosody_bind_http_port`. SELinux stays in enforcing mode.
4. File sharing works over HTTPS on port 443, and file links have no port number. Calls go through coturn.
5. The certificate for `inkov.dev` and `*.inkov.dev` is issued with DNS-01 and the Cloudflare API, renewed by the `certbot-renew.timer` timer and imported into Prosody.
6. Login to the server is by SSH key only, Cockpit is disabled, firewalld opens only the required ports, registration on the XMPP server is closed, and coturn does not relay traffic to internal addresses.
7. Every 6-12 months the server is upgraded to the next Fedora release.

### Checklist

- [ ] OS updated, SELinux in enforcing mode, a user from the `wheel` group logs in with a key, password and root login disabled (section 2)
- [ ] Cockpit disabled, dnf5-automatic installs security updates (sections 2.8 and 2.9)
- [ ] DNS records created in DNS only mode and visible with `dig` (section 3)
- [ ] Prosody and `lua-dbi` installed, `prosodyctl about` shows Lua 5.4 and `LuaDBI` (section 4.2)
- [ ] Drop-in with `AmbientCapabilities` created, `prosody_bind_http_port` turned on (section 4.3)
- [ ] firewalld allows `https`, `xmpp-client`, `xmpp-server`, the `cockpit` service removed from the zone (section 5)
- [ ] Certificate issued, `certbot-renew.timer` enabled, `sudo certbot renew --dry-run` passes (section 6)
- [ ] `sudo prosodyctl check config` shows no errors (section 7)
- [ ] `sudo prosodyctl check turn -v` shows `Success!` (section 8)
- [ ] `/var/lib/prosody/prosody.sqlite` created after start, no storage errors in `prosody.err` (section 9.1)
- [ ] Certificate imported, port 443 answers for XMPP and HTTPS, `check certs` and `check connectivity` show no errors (section 9)
- [ ] Messages, files, group chats, calls and federation work (section 10)
- [ ] Backup created and copied off the server (section 11)
- [ ] Date for the move to the next Fedora release planned (section 11.6)

### Security measures

The server is reachable from the internet, so its security depends on regular maintenance. Once a month update the packages and check the certificate expiry date (`sudo certbot certificates`). Do not disable SELinux, do not expose ports 5280 and 9090, do not enable password login for SSH, and do not disable `s2s_secure_auth` and `c2s_require_encryption`. Keep the SSH key, the files from `/root/.secrets` and the backups in a safe place. Move to the next Fedora release before support for the current one ends.

### Documentation

- Prosody: https://prosody.im/doc
- Installing Prosody on Fedora: https://prosody.im/download/start
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
- Data storage: https://prosody.im/doc/storage
- mod_storage_sql: https://prosody.im/doc/modules/mod_storage_sql
- Third-party modules: https://prosody.im/doc/installing_modules
- XEP-0368 (Direct TLS): https://xmpp.org/extensions/xep-0368.html
- Fedora Server: https://docs.fedoraproject.org/en-US/fedora-server/
- Fedora release life cycle: https://docs.fedoraproject.org/en-US/releases/lifecycle/
- Upgrading Fedora to the next release: https://docs.fedoraproject.org/en-US/quick-docs/upgrading-fedora-offline/
- SELinux in Fedora: https://docs.fedoraproject.org/en-US/quick-docs/selinux-getting-started/
- firewalld: https://docs.fedoraproject.org/en-US/quick-docs/firewalld/, https://firewalld.org/documentation/
- coturn: https://github.com/coturn/coturn
- certbot: https://eff-certbot.readthedocs.io/en/stable/
- certbot plugin for Cloudflare: https://certbot-dns-cloudflare.readthedocs.io/en/stable/
- DNS in Cloudflare: https://developers.cloudflare.com/dns/
- Cloudflare API tokens: https://developers.cloudflare.com/fundamentals/api/get-started/create-token/

---

## 14. Fast connection: SASL2, Bind 2, FAST (optional)

Do this section after section 10, when the server is already running. It speeds up connection and reconnection for mobile clients. The server works without it.

### 14.1. What it does

- **XEP-0388 (SASL2)** - a new login protocol that replaces SASL from RFC 6120. The client sends its identifier, the software name and the device. After a successful login the stream is not restarted, as it is with regular SASL, and extra actions can be added to the login request itself.
- **XEP-0386 (Bind 2)** - resource binding inside the SASL2 request. In the same request the client enables Carbons and reports its CSI state. The `mod_sasl2_sm` module adds enabling and resuming Stream Management (XEP-0198) to the request.
- **XEP-0484 (FAST)** - token login. After the first password login the client gets a token and uses it for later logins. The token is valid for 21 days and is replaced with a new one once a day. A password change invalidates all issued tokens.

XEP-0388 and XEP-0386 have Stable status. They are not in the Prosody 13 core: they are implemented in third-party modules from the prosody-modules repository, which have Beta status.

Number of round trips to the server when reconnecting with session resumption, after the TCP connection is established:

| Stage | Regular SASL | SASL2, Bind 2, SM | SASL2, Bind 2, SM, FAST |
|---|---|---|---|
| TLS and stream opening | 2 | 2 | 2 |
| Login | 2 (SCRAM) | 2 (SCRAM) | 1 (token) |
| Stream restart | 1 | - | - |
| Session resumption | 1 | inside login | inside login |
| Total | about 6 | about 4 | about 3 |

On a mobile network with 150-300 ms latency, reconnection gets shorter by roughly half a second to a second. This is noticeable with frequent network changes and on iOS: after a push notification the app has little time to connect and fetch messages.

What does not change: federation, file sharing, calls, OMEMO. Clients without SASL2 support keep logging in with regular SASL: the server offers both.

Support in the clients from section 10.2:

| Client | SASL2, Bind 2, FAST |
|---|---|
| Conversations, Cheogram, Monocles Chat | yes |
| Monal | yes |
| Gajim | SASL2 is supported, Bind 2 and FAST support not confirmed |
| Dino | in development, not confirmed in a stable release |

### 14.2. Risks and when to enable

Risks:

1. **Third-party modules with Beta status.** They are not in the Fedora package, and `dnf upgrade` does not update them. The guide was tested on revision `9503fcbf014f` of the prosody-modules repository with Prosody 13.0.6. Newer revisions target the next Prosody version and may use features that 13.x does not have.
2. **Major Prosody version change.** When you move to the next Fedora release, Prosody may be updated to version 14. The modules may stop loading, or move into the Prosody core and conflict with the copies in `/usr/local/lib/prosody/modules`. If a module does not load, Prosody logs an error and clients fall back to regular SASL. If a module loads but works incorrectly, login breaks for clients with SASL2 support, which means the main mobile clients. Rollback takes a minute (14.9).
3. **FAST tokens.** A token on a device gives access to the account without a password for up to 21 days. All of a user's tokens can be revoked with a password change (`sudo prosodyctl passwd`), and the tokens of a single device with the command from 14.6.
4. **Diagnostics.** `prosodyctl check` does not check these modules. Login errors are investigated through the log in `debug` mode.

The modules are installed by copying files, not with `prosodyctl install`. The Prosody installer uses luarocks, and the `luarocks` package in Fedora requires the gcc compiler, `zip` and `unzip`, and installs the Lua 5.1 headers (`compat-lua-devel`) by default. The server does not need these packages.

When to enable:

- A few users, a stable connection, mostly desktop clients: the gain is barely noticeable, `smacks` already resumes the session without losing messages.
- Conversations and Monal as the main clients, frequent network changes, iOS: worth enabling.
- You need a list of devices that have access to the account: enable it together with `mod_client_management` (14.6).

### 14.3. Installing the modules

The modules are copied to a separate directory, `/usr/local/lib/prosody/modules`. Files in it get the SELinux label `lib_t`, which Prosody can read. The `.hg_archival.txt` file from the archive stores the revision, and `prosodyctl about` shows it.

```bash
# prosody-modules revision this guide was tested on
REV=9503fcbf014f
# Modules: SASL2, Bind 2, Stream Management inside login, FAST
MODS="mod_sasl2 mod_sasl2_bind2 mod_sasl2_sm mod_sasl2_fast"
tmp=$(mktemp -d)
curl -fsSL "https://hg.prosody.im/prosody-modules/archive/$REV.tar.gz" | tar -xz -C "$tmp" --strip-components=1
# Compatibility from the README: expect a "13 ... Works" line for each module
grep -H -E '^ *13 ' $(for m in $MODS; do echo "$tmp/$m/README.md"; done)
# Directory for third-party modules, then copy
sudo install -d -m 755 /usr/local/lib/prosody/modules
for m in $MODS; do sudo cp -r "$tmp/$m" /usr/local/lib/prosody/modules/; done
sudo cp "$tmp/.hg_archival.txt" /usr/local/lib/prosody/modules/
rm -rf "$tmp"
# Default SELinux labels and check
sudo restorecon -R /usr/local/lib/prosody
ls -Z /usr/local/lib/prosody/modules
```

**Expected:** a line with `13` and `Works` for each module (the README of `mod_sasl2_sm` and `mod_sasl2_fast` says `Work`), four `mod_sasl2*` subdirectories with the `lib_t` label, for example `unconfined_u:object_r:lib_t:s0 mod_sasl2`.

- The files are copied with `cp`, not `mv`: files moved from `/tmp` would keep the label of the temporary directory, and Prosody could not read them.
- If `hg.prosody.im` is unreachable from the VPS, download the archive `https://hg.prosody.im/prosody-modules/archive/9503fcbf014f.tar.gz` on another computer, copy it to the VPS with `scp` and run the commands starting from `tar -xz`, with the file as input: `tar -xzf <archive> -C "$tmp" --strip-components=1`.

### 14.4. Configuration

```bash
# Backup and rollback: sudo cp /etc/prosody/prosody.cfg.lua.before-sasl2 /etc/prosody/prosody.cfg.lua
sudo cp /etc/prosody/prosody.cfg.lua /etc/prosody/prosody.cfg.lua.before-sasl2
sudo nano /etc/prosody/prosody.cfg.lua
```

In the global part, after the line `certificates = "/etc/pki/prosody"`, add:

```lua
-- Third-party modules (section 14): SASL2, Bind 2, FAST
plugin_paths = { "/usr/local/lib/prosody/modules" }
```

In `modules_enabled`, after the line `"net_multiplex";`, add:

```lua
    -- Fast connection (section 14)
        "sasl2"; -- SASL2 login (XEP-0388)
        "sasl2_bind2"; -- resource binding inside login (XEP-0386)
        "sasl2_sm"; -- session resumption inside login (XEP-0198)
        "sasl2_fast"; -- token login (XEP-0484)
```

The FAST token lifetime can be shortened (optional). Add the line to the global part next to `plugin_paths`:

```lua
-- FAST token lifetime: 7 days instead of 21. A client that has not connected for longer asks for the password again
sasl2_fast_token_ttl = 7 * 86400
```

Check and restart:

```bash
sudo prosodyctl check config
sudo systemctl restart prosody
```

**Expected:** `All checks passed, congratulations!`. New modules and `plugin_paths` are loaded only at startup, so you need `restart`, not `reload`. Clients disconnect during the restart and then connect again.

### 14.5. Verification

```bash
# The module directory is loaded and the revision is detected
sudo prosodyctl about | grep prosody-modules
# Module load errors: expect empty output (-s: the file may not exist)
sudo grep -s -i sasl2 /var/log/prosody/prosody.err
```

**Expected:**

```text
  /usr/local/lib/prosody/modules - prosody-modules rev: 9503fcbf014f
```

Check on the server or on a local computer that Prosody offers SASL2, Bind 2, FAST and Stream Management:

```bash
(printf "<?xml version='1.0'?><stream:stream to='inkov.dev' from='admin@inkov.dev' xmlns='jabber:client' xmlns:stream='http://etherx.jabber.org/streams' version='1.0'>"; sleep 3) \
  | timeout 6 openssl s_client -connect xmpp.inkov.dev:443 -servername inkov.dev -alpn xmpp-client -quiet 2>/dev/null \
  | grep -o -E 'urn:xmpp:(sasl:2|bind:0|fast:0|sm:3)' | sort -u
```

**Expected:**

```text
urn:xmpp:bind:0
urn:xmpp:fast:0
urn:xmpp:sasl:2
urn:xmpp:sm:3
```

The `from` attribute in the stream header is needed for the FAST check: Prosody offers FAST only if the client has given its address. Clients with SASL2 support send it themselves. If there is no `urn:xmpp:sasl:2` line, the modules are not loaded: check `plugin_paths`, the `modules_enabled` list and whether Prosody was restarted.

After the Prosody restart, Conversations and Monal switch to SASL2 on their next connection, no client setting is needed. The device list (14.6) shows which login method a client uses.

### 14.6. Device list (optional)

The `mod_client_management` module lists the clients that have access to the account and revokes access. It uses the client data from SASL2 and the FAST tokens, so it requires `mod_sasl2_fast`.

```bash
REV=9503fcbf014f
tmp=$(mktemp -d)
curl -fsSL "https://hg.prosody.im/prosody-modules/archive/$REV.tar.gz" | tar -xz -C "$tmp" --strip-components=1
sudo cp -r "$tmp/mod_client_management" /usr/local/lib/prosody/modules/
rm -rf "$tmp"
sudo restorecon -R /usr/local/lib/prosody
sudo nano /etc/prosody/prosody.cfg.lua
```

In `modules_enabled`, after the line `"sasl2_fast";`, add:

```lua
        "client_management"; -- list of clients with access to the account
```

```bash
sudo prosodyctl check config && sudo systemctl restart prosody
# Clients of the account
sudo prosodyctl shell user clients admin@inkov.dev
```

**Expected** (example):

```text
ID                    | Software              |  First seen |  Last seen |    Expires | Authentication
------------------------------------------------------------------------------------------------------
client/d4565fa7-4d72… | Conversations         |    09:15:58 |   09:16:05 |            | connected, fast, password
------------------------------------------------------------------------------------------------------
OK: 1 clients
```

In the `Authentication` column: `connected` - the client has a session, `password` - the client logged in with a password, `fast` - the client has a valid FAST token. Clients without SASL2 are identified in the list only by resource and may be shown inaccurately.

Revoke a device's access by the software name from the `Software` column:

```bash
sudo prosodyctl shell user revoke_client admin@inkov.dev software/Conversations
```

The command closes the client's session and revokes its FAST tokens. If the client has logged in with a password at least once, the command also prints `Error: Password reset required`: the password is still saved on the device, and the client can log in again. In that case change the password with `sudo prosodyctl passwd admin@inkov.dev`, which also revokes all FAST tokens of the account. The first login is always done with a password, so for a lost device the reliable option is a password change. The module is mainly useful for the device list.

### 14.7. Password change and tokens

Prosody rejects a FAST token issued before a password change with the `credentials-expired` error. After `sudo prosodyctl passwd` all of the user's devices ask for the new password on the next connection. This behavior was tested on Prosody 13.0.6 and revision `9503fcbf014f`.

### 14.8. Updating the modules

Update the modules when you move to a new Prosody version or when the modules get fixes. Before updating, check the compatibility table in each module's README.

```bash
# New revision: an ID from https://hg.prosody.im/prosody-modules/ or tip for the latest one
REV=<NEW_REVISION>
MODS="mod_sasl2 mod_sasl2_bind2 mod_sasl2_sm mod_sasl2_fast"
# Add mod_client_management if it is installed (14.6)
tmp=$(mktemp -d)
curl -fsSL "https://hg.prosody.im/prosody-modules/archive/$REV.tar.gz" | tar -xz -C "$tmp" --strip-components=1
# Compatibility with the installed Prosody version
grep -H -A 6 -i 'compatibility' $(for m in $MODS; do echo "$tmp/$m/README.md"; done)
# Copy of the current modules for rollback
sudo cp -a /usr/local/lib/prosody/modules /usr/local/lib/prosody/modules.bak-$(date +%F)
# Replace each module directory completely, so files removed in the new revision do not stay behind
for m in $MODS; do
  sudo rm -rf "/usr/local/lib/prosody/modules/$m"
  sudo cp -r "$tmp/$m" /usr/local/lib/prosody/modules/
done
sudo cp "$tmp/.hg_archival.txt" /usr/local/lib/prosody/modules/
rm -rf "$tmp"
sudo restorecon -R /usr/local/lib/prosody
sudo prosodyctl check config && sudo systemctl restart prosody
```

Then repeat the checks from 14.5. Rollback to the previous revision:

```bash
sudo rm -rf /usr/local/lib/prosody/modules
sudo cp -a /usr/local/lib/prosody/modules.bak-<date> /usr/local/lib/prosody/modules
sudo systemctl restart prosody
```

### 14.9. Rollback

```bash
# Configuration from before section 14
sudo cp /etc/prosody/prosody.cfg.lua.before-sasl2 /etc/prosody/prosody.cfg.lua
sudo prosodyctl check config && sudo systemctl restart prosody
# Module files (optional)
sudo rm -rf /usr/local/lib/prosody
```

If you changed anything else in the configuration after section 14, do not restore the backup. Instead, delete the `plugin_paths` line and the `sasl2*` and `client_management` lines with `sudo nano`. FAST tokens stay in the database and are not used. Clients go back to regular SASL on their next connection. If a client does not connect after the rollback, log in to the account in that client again.

### 14.10. Troubleshooting

| Symptom | Likely cause | How to check | How to fix |
|---|---|---|---|
| `prosody.err` shows a load error for the `sasl2` module (`module not found`) | No `plugin_paths`, the files are in a different directory, or the SELinux label is wrong | `sudo prosodyctl about`, `ls -Z /usr/local/lib/prosody/modules` | Check `plugin_paths`, run `sudo restorecon -R /usr/local/lib/prosody` |
| The stream features have no `urn:xmpp:sasl:2` | The modules are not in `modules_enabled`, or Prosody was not restarted | The check from 14.5 | Add the modules (14.4), `sudo systemctl restart prosody` |
| No `urn:xmpp:fast:0`, the other lines are there | `sasl2_fast` is not loaded, or the stream header has no `from` | The check from 14.5 with the `from` attribute | Add `"sasl2_fast";`, restart Prosody |
| After the modules are enabled, Conversations or Monal cannot log in while other clients can | A bug in the modules or an incompatible revision | `debug` in the `log` block, `sasl2` entries in `prosody.log` | Rollback (14.9) or the previous revision (14.8) |
| `revoke_client` prints `Error: Password reset required` | The client logged in with a password | `sudo prosodyctl shell user clients user@inkov.dev` | Change the password: `sudo prosodyctl passwd user@inkov.dev` |

Documentation:

- mod_sasl2: https://modules.prosody.im/mod_sasl2
- mod_sasl2_bind2: https://modules.prosody.im/mod_sasl2_bind2
- mod_sasl2_sm: https://modules.prosody.im/mod_sasl2_sm
- mod_sasl2_fast: https://modules.prosody.im/mod_sasl2_fast
- mod_client_management: https://modules.prosody.im/mod_client_management
- XEP-0388 (SASL2): https://xmpp.org/extensions/xep-0388.html
- XEP-0386 (Bind 2): https://xmpp.org/extensions/xep-0386.html
- XEP-0484 (FAST): https://xmpp.org/extensions/xep-0484.html
- Prosody blog post about FAST: https://blog.prosody.im/fast-auth/
