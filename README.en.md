# xmpp-server-guides

[Русская версия](README.md)

Guides for installing and configuring XMPP servers on Ubuntu LTS and Fedora Server.

This repository collects practical guides for deploying XMPP servers alongside reverse proxies, control panels and other infrastructure. Each guide is a single Markdown file, tested on a specific OS release with specific package versions.

The guides target VPS hosting in Russia. Because of that they include reachability checks for package mirrors (APT or dnf), the Prosody repository, Let's Encrypt and the Cloudflare API, plus fallbacks for when one of them is blocked from the provider's network. On a VPS in another country these checks pass and the rest of the setup is the same.

## Guides

| Guide | OS | Use case |
|---|---|---|
| [Prosody on a VPS with 3x-ui](GUIDE_XMPP_Prosody_Ubuntu_22.04_HAProxy_3x-ui.en.md) | Ubuntu 22.04 | The server already runs 3x-ui (Xray), port 443 is taken |
| [Prosody on a clean VPS](GUIDE_XMPP_Prosody_Ubuntu_24.04.en.md) | Ubuntu 24.04 | Dedicated XMPP server |
| [Prosody on a clean VPS](GUIDE_XMPP_Prosody_Ubuntu_26.04.en.md) | Ubuntu 26.04 | Dedicated XMPP server |
| [Prosody on a clean VPS](GUIDE_XMPP_Prosody_Fedora_44.en.md) | Fedora Server 44 | Dedicated XMPP server, SELinux in enforcing mode |

The Russian originals have the same names without the `.en` suffix.

### Which one to pick

- **Your VPS runs 3x-ui.** Use the Ubuntu 22.04 guide with 3x-ui. It leaves the 3x-ui configuration alone: Prosody accepts clients on port 5222 and other servers on 5269, file sharing runs over HTTPS on 5281. Appendix A shows how to serve XMPP and file sharing on port 443 next to REALITY: Xray hands connections that fail the REALITY check to HAProxy, and HAProxy routes them by SNI without decrypting them.
- **Dedicated server on Ubuntu.** Use the Ubuntu 24.04 or 26.04 guide, depending on your release. Prosody owns port 443 and serves both client connections (Direct TLS, XEP-0368) and HTTPS for file sharing on it. Data is stored in SQLite. The guide also covers basic server setup: a sudo user, SSH key-only login and ufw. The 26.04 guide differs from 24.04 in a few places: the Prosody repository is added by hand, time sync is done by chrony instead of systemd-timesyncd, and sudo is sudo-rs.
- **Dedicated server on Ubuntu 22.04.** The clean VPS guides do not work on 22.04: its `lua-dbi-sqlite3` driver is built only for Lua 5.1-5.3, and Prosody 13 runs on Lua 5.4. Use the 3x-ui guide instead (it keeps the default Prosody storage) and skip the 3x-ui and Xray steps.
- **Dedicated server on Fedora.** Use the Fedora Server 44 guide. The layout matches the Ubuntu 24.04 and 26.04 guides: Prosody on port 443, data in SQLite. All packages come from the Fedora repositories, no third-party repositories are needed. SELinux stays in enforcing mode: port 443 is allowed with a policy boolean, and the service gets the right to bind ports below 1024 through a systemd drop-in. The firewall is firewalld, and the Cockpit web console is turned off. A Fedora release is supported for about 13 months, so every 6-12 months the server has to be upgraded to the next release. If you want to upgrade the OS less often, pick Ubuntu LTS. Only this guide has the optional section 14: the SASL2, Bind 2 and FAST modules, which make mobile clients reconnect faster.

## What you end up with

Server features are the same in all guides. The port layout and system preparation differ.

- Prosody 13: on Ubuntu from the official packages.prosody.im repository, on Fedora from the distribution repository.
- Addresses like `user@example.org`. The website on the apex domain stays where it is (GitHub Pages in the examples). Clients and other servers find the XMPP server through SRV records.
- A wildcard Let's Encrypt certificate, issued via DNS-01 and the Cloudflare API, renewed automatically and imported into Prosody.
- Server-side message archive (MAM), file sharing, group chats.
- Calls through coturn (STUN/TURN).
- Modules for mobile clients: session resumption after a network change, message copies on all devices, push notifications.
- Closed registration: accounts are created by the admin or through invites.
- Mandatory encryption for clients and servers, certificate checks for remote servers.
- Sections on backups, updates and troubleshooting.

## Requirements

- A VPS with the matching OS release (Ubuntu LTS or Fedora Server) and SSH access.
- A domain with DNS hosted on Cloudflare. With another DNS provider, create the records from section 3 in its panel, and for the certificate in section 6 use the matching certbot DNS plugin or the manual DNS-01 option.
- Access to the VPS console in the provider's panel (VNC or similar) in case an SSH or firewall change locks you out.
- An XMPP client. Section 10.2 of each guide lists clients for Android, iOS, macOS, Linux and Windows.

## How to read the guides

- The examples use the domain `inkov.dev` and the server `xmpp.inkov.dev`. The easiest way to adapt a guide is to make a copy with your own domain:

    ```bash
    sed 's/inkov\.dev/example.org/g' GUIDE_XMPP_Prosody_Ubuntu_24.04.en.md > my-guide.md
    ```

    After that, check the account names as well: the `admins` list in the Prosody config and the examples in section 10.1.
- Values in angle brackets (`<IP_VPS>`, `<EMAIL>`, `<CF_API_TOKEN>` and so on) are placeholders. Section 1.5 of each guide lists them with explanations.
- Commands run on the VPS unless stated otherwise.
- Most steps end with an "Expected" block that shows the output you should get. If your output differs, find the cause before moving on. Section 12 has a table of common problems.
- Most steps that change the system come with rollback commands.

## Tested versions

| Guide | OS | Prosody | coturn | certbot | Other |
|---|---|---|---|---|---|
| 22.04, VPS with 3x-ui | Ubuntu 22.04.5 LTS | 13.0.7 | 4.5.2 | 1.21 | HAProxy 2.4 and Xray 26.3.27 (Appendix A) |
| 24.04, clean VPS | Ubuntu 24.04.5 LTS | 13.0.7 | 4.6.1 | 2.9.0 | lua-dbi-sqlite3 0.7.2 |
| 26.04, clean VPS | Ubuntu 26.04.1 LTS | 13.0.7 | 4.6.1 | 4.0.0 | lua-dbi-sqlite3 0.7.2 |
| Fedora 44, clean VPS | Fedora Server 44 | 13.0.6 | 4.18.0 | 5.8.0 | lua-dbi 0.7.5, prosody-modules revision 9503fcbf014f (section 14) |

## Repository layout

One guide per file, with the English version next to it under the `.en.md` suffix. File names follow the pattern `GUIDE_<protocol>_<server>_<OS>_<OS version>[_<extra stack>].md`. New guides should use the same pattern.

## Feedback

Report broken commands or outdated steps in the issues. Fixes are welcome as merge requests. If you change a guide, update both language versions.
