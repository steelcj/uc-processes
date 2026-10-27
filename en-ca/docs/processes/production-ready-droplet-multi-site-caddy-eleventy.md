---
title: Production-ready Droplet for multi-site Caddy and Eleventy hosting
description: A lightweight, secure, rebuildable DigitalOcean Droplet on Debian 13 that serves multiple Eleventy-built static websites through Caddy.
version: 0.1.0
date: 2026-10-24
status: draft
author: Christopher Steel
lang: en-CA
---

# Production-ready Droplet for multi-site Caddy and Eleventy hosting

## Purpose and scope

This document describes a complete, reproducible setup for a single DigitalOcean Droplet that hosts several static websites built with Eleventy and served by Caddy. It replaces the general-purpose baseline in DigitalOcean's own production-ready Droplet guide <a id="cite-digitalocean-2026b"></a>([DigitalOcean, 2026b](#ref-digitalocean-2026b)) with a configuration aimed specifically at static hosting, and it corrects several problems in that guide, summarized near the end.

The setup has four goals. The server should be small, with nothing installed that the workload does not need. It should be secure by default, with key-only SSH, no root login, and two layers of firewall. It should be fast for visitors, including visitors on low or intermittent bandwidth, by serving precompressed files over HTTP/2 and HTTP/3. And it should be rebuildable and portable, so that the whole server is described by one cloud-init file and can be recreated on DigitalOcean, another provider, or a local Incus host without relying on provider-specific backups.

The document is a draft. Every command and file here should be validated in a disposable environment before it is used for client sites.

## Architecture overview

The design separates building from serving.

- Eleventy runs on the workstation or in CI, never on the server. The build output in `_site` is precompressed (gzip, Brotli and Zstandard) at build time.
- The output is pushed to the Droplet with `rsync` over SSH, as a dedicated `deploy` user whose key can only write into `/srv/sites`.
- Caddy serves each site from `/srv/sites/<domain>`, obtains and renews TLS certificates automatically, and prefers the precompressed files when the visitor's browser accepts them.
- A DigitalOcean Cloud Firewall filters traffic before it reaches the Droplet, and an nftables ruleset on the Droplet mirrors it, so the protection travels with the server if it moves.
- The server itself holds no irreplaceable state. Its configuration lives in version control as `cloud-config.yaml`, the per-site Caddy files live in an infrastructure repository, and the site content lives in each site's repository.

Keeping Node.js and the npm dependency tree off the production server removes the largest attack surface a static site normally carries, and keeps the server's package list short enough to audit.

## Design decisions

### Operating system

DigitalOcean currently offers Debian 13 (`debian-13-x64`) alongside Ubuntu, Fedora, AlmaLinux, CentOS Stream and Rocky Linux images <a id="cite-digitalocean-nd-b"></a>([DigitalOcean, n.d.-b](#ref-digitalocean-nd-b)). This setup uses Debian 13 "trixie", released on August 9, 2025, and supported for five years through the Debian Security and Long Term Support teams <a id="cite-debian-2025"></a>([Debian, 2025](#ref-debian-2025)). Debian has a small default footprint, does not ship snapd, changes slowly within a release, and is supported directly by Caddy's official package repository.

For static file serving the operating system has little effect on visitor-facing speed. Speed comes from Caddy, precompression, caching headers and the network. Debian is chosen for its stability, small attack surface and portability, not for raw performance.

### Rebuild instead of restore

DigitalOcean's paid backups take disk images of the whole Droplet. For a static host, those images mostly capture things that can be recreated: the operating system, packages and copies of site files that already exist in version control. This setup leaves backups off and treats the server as rebuildable from `cloud-config.yaml`, the infrastructure repository and the site repositories. A DigitalOcean disk image is also the least portable artifact you could keep, while a cloud-init file runs on almost any cloud and on Incus.

### Two firewalls

The Cloud Firewall is enforced on DigitalOcean's network, is free, and keeps unwanted traffic from reaching the Droplet at all. It is also specific to DigitalOcean. The nftables ruleset on the Droplet provides the same policy in a form that moves with the server, and still protects it if the Cloud Firewall is detached by mistake.

### Two users

`sysadmin` is the only interactive account, with sudo protected by a password. `deploy` has no password, no shell access and no sudo, and its SSH key is locked to a restricted `rsync` command that can only write under `/srv/sites`. A leaked deploy key can overwrite website files, but cannot read the server, run commands or escalate.

## Prerequisites

On the workstation:

- `doctl`, authenticated against the DigitalOcean account
- an Ed25519 key pair for `sysadmin`, for example `~/.ssh/id_ed25519`
- a separate Ed25519 key pair for deployment, for example `~/.ssh/deploy_ed25519`, created with `ssh-keygen -t ed25519 -f ~/.ssh/deploy_ed25519 -C deploy`
- `mkpasswd` (in the `whois` package on Debian and Ubuntu derivatives) to hash the `sysadmin` sudo password
- `brotli`, `zstd`, `gzip` and `rsync` for the build and deploy script
- optionally, Incus, to test the cloud-init file locally before using it

In DNS, control over the zone of each domain to be hosted, so that A and AAAA records can point at the Droplet.

Also upload at least one SSH public key to the DigitalOcean account and select it when creating the Droplet, even though this setup installs its own keys. When no key is specified, DigitalOcean emails a temporary root password instead.

## Server layout

| Path | Owner | Purpose |
| --- | --- | --- |
| `/etc/caddy/Caddyfile` | root | Global options and the shared `static-site` snippet |
| `/etc/caddy/sites/<domain>.caddy` | root | One file per hosted site |
| `/srv/sites/` | deploy | Root of all published site content |
| `/srv/sites/<domain>/` | deploy | Eleventy output for one site, created by the first deploy |
| `/etc/ssh/sshd_config.d/00-hardening.conf` | root | SSH policy |
| `/etc/nftables.conf` | root | Host firewall ruleset |
| `/var/lib/caddy/` | caddy | Caddy's data directory, including certificates, in the Debian package's default layout |

Files under `/srv/sites` are published with directory mode 755 and file mode 644, which lets the `caddy` service account read them without belonging to the `deploy` group.

## The cloud firewall

DigitalOcean Cloud Firewalls support only ICMP, TCP and UDP, and allow up to 50 combined inbound and outbound rules per firewall <a id="cite-digitalocean-2026a"></a>([DigitalOcean, 2026a](#ref-digitalocean-2026a)). They are applied by tag here, so any future Droplet tagged `web-static` inherits the same rules. They are separate from any firewall running inside the Droplet <a id="cite-digitalocean-nd-a"></a>([DigitalOcean, n.d.-a](#ref-digitalocean-nd-a)).

The inbound rules allow SSH only from the administrator's address, HTTP and HTTPS from anywhere on both IPv4 and IPv6, UDP 443 for HTTP/3, and ICMP so that path MTU discovery and diagnostics keep working. HTTP/3 runs over UDP, and when UDP 443 is blocked, browsers silently fall back to HTTP/2 over TCP <a id="cite-geniar-2026"></a>([Geniar, 2026](#ref-geniar-2026)).

Replace `203.0.113.10/32` with your own address, and add a second `protocol:tcp,ports:22` rule for any other fixed address you administer from.

```shell
doctl compute firewall create --name "web-static" \
    --tag-names web-static \
    --inbound-rules "protocol:tcp,ports:22,address:203.0.113.10/32 protocol:tcp,ports:80,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:443,address:0.0.0.0/0,address:::/0 protocol:udp,ports:443,address:0.0.0.0/0,address:::/0 protocol:icmp,address:0.0.0.0/0,address:::/0" \
    --outbound-rules "protocol:icmp,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:all,address:0.0.0.0/0,address:::/0 protocol:udp,ports:all,address:0.0.0.0/0,address:::/0"
```

Restricting SSH by address has one consequence for deployment: hosted CI runners usually have changing addresses and will be refused. Either deploy from a machine with a fixed address, add the CI provider's published ranges, or put SSH behind WireGuard or Tailscale and remove the public port 22 rule entirely. If your own address changes and you are locked out, the DigitalOcean Droplet console still provides access.

## The cloud-init file

The whole server is defined in one `#cloud-config` file. Unlike the bash script in DigitalOcean's guide, it is declarative YAML that cloud-init on any provider understands.

Before using it, make three substitutions:

- Replace both `ssh-ed25519 AAAA...` placeholders with the contents of `~/.ssh/id_ed25519.pub` and `~/.ssh/deploy_ed25519.pub`.
- Replace `REPLACE_WITH_YESCRYPT_HASH` with the output of `mkpasswd --method=yescrypt`, which prompts for the `sysadmin` sudo password.
- Replace `webmaster@example.org` with the address Caddy should give the certificate authority for expiry notices.

The password hash travels in the Droplet's user data, which is readable from inside the Droplet through the metadata service. The cloud-init project's own examples warn that a weak password behind such a hash can be cracked in seconds <a id="cite-cloud-init-nd"></a>([cloud-init, n.d.](#ref-cloud-init-nd)), so use a long, unique password generated by a password manager.

```yaml
#cloud-config
# Static web host: Debian 13, Caddy, multi-site Eleventy output.
# Version 0.1.0

hostname: web-01

package_update: true
package_upgrade: true
packages:
  - nftables
  - unattended-upgrades
  - rsync
  - curl
  - gnupg
  - debian-keyring
  - debian-archive-keyring
  - apt-transport-https

ssh_pwauth: false
disable_root: true

users:
  - name: sysadmin
    gecos: Server administrator
    groups: [sudo]
    shell: /bin/bash
    lock_passwd: false
    hashed_passwd: "REPLACE_WITH_YESCRYPT_HASH"
    ssh_authorized_keys:
      - "ssh-ed25519 AAAA... sysadmin@workstation"
  - name: deploy
    gecos: Static site deployment
    shell: /bin/sh
    lock_passwd: true
    ssh_authorized_keys:
      - 'command="rrsync -wo /srv/sites",restrict ssh-ed25519 AAAA... deploy@workstation'

write_files:
  - path: /etc/ssh/sshd_config.d/00-hardening.conf
    owner: root:root
    permissions: "0644"
    content: |
      PermitRootLogin no
      PasswordAuthentication no
      KbdInteractiveAuthentication no
      AuthenticationMethods publickey
      AllowUsers sysadmin deploy
      X11Forwarding no

  - path: /etc/nftables.conf
    owner: root:root
    permissions: "0755"
    defer: true
    content: |
      #!/usr/sbin/nft -f
      flush ruleset

      table inet filter {
        chain input {
          type filter hook input priority filter; policy drop;
          ct state established,related accept
          ct state invalid drop
          iif "lo" accept
          meta l4proto icmp accept
          meta l4proto ipv6-icmp accept
          tcp dport 22 accept
          tcp dport { 80, 443 } accept
          udp dport 443 accept
        }
        chain forward {
          type filter hook forward priority filter; policy drop;
        }
        chain output {
          type filter hook output priority filter; policy accept;
        }
      }

  - path: /etc/apt/apt.conf.d/20auto-upgrades
    owner: root:root
    permissions: "0644"
    defer: true
    content: |
      APT::Periodic::Update-Package-Lists "1";
      APT::Periodic::Unattended-Upgrade "1";

  - path: /root/bootstrap/Caddyfile
    owner: root:root
    permissions: "0644"
    content: |
      {
      	email webmaster@example.org
      }

      (static-site) {
      	root * /srv/sites/{args[0]}
      	encode zstd gzip
      	file_server {
      		precompressed zstd br gzip
      	}
      	header {
      		Strict-Transport-Security "max-age=31536000"
      		X-Content-Type-Options "nosniff"
      		Referrer-Policy "strict-origin-when-cross-origin"
      		-Server
      	}
      	@immutable path_regexp \.[0-9a-f]{8,}\.(css|js|mjs|woff2|avif|webp|png|jpg|svg)$
      	header @immutable Cache-Control "public, max-age=31536000, immutable"
      	@pages path / *.html */
      	header @pages Cache-Control "public, max-age=300, must-revalidate"
      	handle_errors {
      		rewrite * /{err.status_code}.html
      		file_server
      	}
      }

      import /etc/caddy/sites/*.caddy

runcmd:
  - [systemctl, enable, --now, nftables]
  - [sh, -c, "sshd -t && systemctl restart ssh"]
  - [install, -d, -o, deploy, -g, deploy, -m, "0755", /srv/sites]
  - [sh, -c, "curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg"]
  - [sh, -c, "curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' > /etc/apt/sources.list.d/caddy-stable.list"]
  - [chmod, o+r, /usr/share/keyrings/caddy-stable-archive-keyring.gpg, /etc/apt/sources.list.d/caddy-stable.list]
  - [apt-get, update]
  - [apt-get, install, -y, caddy]
  - [install, -d, -m, "0755", /etc/caddy/sites]
  - [install, -m, "0644", /root/bootstrap/Caddyfile, /etc/caddy/Caddyfile]
  - [sh, -c, "caddy validate --config /etc/caddy/Caddyfile && systemctl reload caddy"]
```

### What each part does

**Ordering.** cloud-init writes ordinary `write_files` entries before it installs packages. The nftables ruleset and the `20auto-upgrades` file are marked `defer: true`, so they are written after `nftables` and `unattended-upgrades` are installed and cannot be replaced by those packages' own defaults. `runcmd` runs last, after all files are in place.

**Users and SSH.** `ssh_pwauth: false` turns off password authentication through cloud-init's own SSH configuration, and `disable_root: true` blocks root logins with the keys DigitalOcean injects. The `00-hardening.conf` drop-in states the policy explicitly. In OpenSSH, the first value obtained for each keyword is the one used <a id="cite-openssh-nd"></a>([OpenSSH, n.d.](#ref-openssh-nd)), and Debian's `sshd_config` includes `/etc/ssh/sshd_config.d/*.conf` at the top of the file. A drop-in named `00-` is therefore read before cloud-init's own `50-cloud-init.conf` and before anything in the main file, so editing the main file with `sed`, as DigitalOcean's script does, can be silently overridden. The `runcmd` step checks the configuration with `sshd -t` before restarting the `ssh` service.

**The deploy key.** The `command="rrsync -wo /srv/sites",restrict` prefix forces every connection with that key to run `rrsync`, which only permits an `rsync` transfer confined to the named directory, and `-wo` allows only writing to it <a id="cite-rsync-2025"></a>([The rsync project, 2025](#ref-rsync-2025)). The `restrict` option removes port forwarding, agent forwarding and terminal allocation. The rrsync manual warns of a potential issue when the restricted user's login shell is bash and the user can modify their own startup files, so `deploy` uses `/bin/sh` and cannot write to its home directory through the restricted transfer.

**Host firewall.** nftables has been Debian's default and recommended firewall framework since Debian 10, and enabling `nftables.service` loads `/etc/nftables.conf` at boot <a id="cite-debian-wiki-nd"></a>([Debian Wiki, n.d.](#ref-debian-wiki-nd)). The ruleset drops all inbound traffic except established connections, loopback, ICMP for both address families, SSH, HTTP, HTTPS and HTTP/3. SSH is open to all addresses here because the Cloud Firewall already restricts it by source. If you want the same restriction inside the Droplet, add an address set to the `tcp dport 22` rule, and keep the Droplet console in mind as the recovery path.

**Automatic security updates.** Installing `unattended-upgrades` is not the same as enabling it on every Debian image, so the setup creates `/etc/apt/apt.conf.d/20auto-upgrades` with both periodic settings explicitly <a id="cite-linuxcapable-nd"></a>([LinuxCapable, n.d.](#ref-linuxcapable-nd)). Debian's default policy applies security updates from Debian's own archives. It does not apply updates from third-party repositories such as Caddy's, which are covered under maintenance below.

**Caddy.** Caddy is installed from the project's official Cloudsmith repository, following the project's Debian instructions <a id="cite-caddy-nd-c"></a>([Caddy, n.d.-c](#ref-caddy-nd-c)), because it tracks current releases and the package runs Caddy as a systemd service named `caddy`. The Caddyfile is staged in `/root/bootstrap` and copied into place after the package is installed, so it does not collide with the package's default file. The final step validates the configuration before reloading Caddy.

## Test locally with Incus

Because the server is defined entirely by cloud-init, it can be tested on a local Incus host before anything is created on DigitalOcean. Debian cloud images run the same cloud-init file.

```shell
incus launch images:debian/13/cloud web-test \
    --config=cloud-init.user-data="$(cat cloud-config.yaml)"
incus exec web-test -- cloud-init status --wait --long
incus exec web-test -- cloud-init schema --system
```

Caddy cannot obtain public certificates for a container without public DNS, but the users, SSH policy, firewall, package installation and Caddy configuration can all be checked this way. Delete the container afterwards with `incus delete --force web-test`.

## Create the Droplet

Choose the region closest to most visitors. For audiences in Quebec and Ontario, `tor1` is the nearest DigitalOcean region. A 1 GB Droplet is more than sufficient for static serving, which is limited by network rather than CPU, and it can be resized later.

```shell
doctl compute droplet create web-01 \
    --image debian-13-x64 \
    --size s-1vcpu-1gb \
    --region tor1 \
    --ssh-keys KEY_FINGERPRINT \
    --user-data-file cloud-config.yaml \
    --tag-names web-static \
    --enable-ipv6 \
    --wait
```

`--enable-ipv6` should be set at creation, because enabling IPv6 later requires manual network configuration and a reboot. Add `--enable-monitoring` if you want DigitalOcean's metrics graphs and alerts; it installs DigitalOcean's agent, which is optional for this workload. Backups are deliberately left off, for the reasons given above.

Retrieve the addresses with `doctl compute droplet get web-01 --format PublicIPv4,PublicIPv6`.

## Verify the first boot

Connect as `sysadmin` and check each layer.

```shell
ssh sysadmin@DROPLET_IPV4

cloud-init status --wait --long
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|authenticationmethods'
sudo nft list ruleset
systemctl status caddy --no-pager
caddy version
command -v rrsync
ls -ld /srv/sites
```

The expected results are: cloud-init reports `done` with no errors; `sshd -T` shows `permitrootlogin no`, `passwordauthentication no`, `kbdinteractiveauthentication no` and `authenticationmethods publickey`; the ruleset matches `/etc/nftables.conf`; Caddy is active; `rrsync` resolves to `/usr/bin/rrsync`; and `/srv/sites` is owned by `deploy` with mode 755.

From the workstation, confirm that root and the deploy key cannot open a shell:

```shell
ssh root@DROPLET_IPV4
ssh -i ~/.ssh/deploy_ed25519 deploy@DROPLET_IPV4
```

The first should be refused. The second should end immediately with an rrsync error rather than a prompt.

## Add a site

Adding a site takes three steps: DNS, a Caddy site file, and a first deploy.

### DNS

Create an A record and an AAAA record for the apex domain and for `www`, pointing at the Droplet's IPv4 and IPv6 addresses. Caddy obtains certificates automatically once the names resolve to the server, and the ACME HTTP challenge requires ports 80 and 443 to be reachable <a id="cite-caddy-docker-nd"></a>([Caddy Docker Maintainers, n.d.](#ref-caddy-docker-nd)), which the firewalls above allow.

### Caddy site file

Each site gets one small file in `/etc/caddy/sites/`, named after the domain. It imports the shared `static-site` snippet and passes the domain as its argument. Caddyfile snippets accept positional arguments, which the snippet reads as `{args[0]}` <a id="cite-caddy-nd-b"></a>([Caddy, n.d.-b](#ref-caddy-nd-b)).

`/etc/caddy/sites/example.org.caddy`:

```caddyfile
example.org {
	import static-site example.org
}

www.example.org {
	redir https://example.org{uri} permanent
}
```

Keep these files in the infrastructure repository and copy them to the server, then validate and reload:

```shell
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

A reload applies the new configuration without dropping existing connections.

### What the shared snippet does

- `root` points Caddy at `/srv/sites/<domain>`.
- `file_server` with `precompressed zstd br gzip` looks for `.zst`, `.br` and `.gz` sidecar files next to the requested file, in that order, and serves the first one the browser accepts with the matching `Content-Encoding` header. When no sidecar exists, it serves the uncompressed file <a id="cite-caddy-nd-a"></a>([Caddy, n.d.-a](#ref-caddy-nd-a)). No CPU is spent compressing at request time.
- `encode zstd gzip` compresses on the fly only for files that have no precompressed sidecar.
- The `header` block sets HSTS, disables MIME sniffing, sets a conservative referrer policy and removes the `Server` header. HSTS is set without `preload`, so it can be withdrawn if a domain ever needs to stop serving HTTPS.
- The `@immutable` matcher gives a one-year immutable cache lifetime to files whose names contain a content hash, such as `main.3f9a1c2b.css`. Eleventy does not fingerprint assets by default, so this rule only takes effect if the build adds hashes. Files without hashes fall through to normal caching.
- The `@pages` matcher gives HTML a short cache lifetime with revalidation, so content changes appear within minutes.
- `handle_errors` serves `404.html` and similar pages from the site's own output, so each Eleventy site should produce a `404.html`.

Access logging is off, which is Caddy's default. Errors still go to the systemd journal, readable with `journalctl -u caddy`.

## Build and deploy from Eleventy

The deploy script runs on the workstation or on a CI runner whose address the Cloud Firewall allows. It builds the site, writes precompressed sidecars for every text file, and pushes the result.

`deploy.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SITE="${1:?usage: deploy.sh <domain>}"
HOST="${DEPLOY_HOST:?set DEPLOY_HOST to the Droplet address or hostname}"
KEY="${DEPLOY_KEY:-$HOME/.ssh/deploy_ed25519}"
OUT="_site"

rm -rf "$OUT"
npx @11ty/eleventy

find "$OUT" -type f \( -name '*.html' -o -name '*.css' -o -name '*.js' -o -name '*.mjs' \
    -o -name '*.svg' -o -name '*.json' -o -name '*.xml' -o -name '*.txt' -o -name '*.webmanifest' \) \
    -print0 |
while IFS= read -r -d '' f; do
    gzip -9 -k -f "$f"
    brotli -q 11 -k -f "$f"
    zstd -19 -q -k -f "$f"
done

rsync -rlptz --delete-after --delay-updates --chmod=D755,F644 \
    -e "ssh -i $KEY" \
    "$OUT"/ "deploy@${HOST}:${SITE}/"
```

Run it from the site's repository as `DEPLOY_HOST=web-01.example.org ./deploy.sh example.org`.

The remote path `example.org/` is relative to `/srv/sites`, because `rrsync` confines the transfer to that directory, and `rsync` creates the site directory on the first deploy. `--delay-updates` places all changed files at the end of the transfer, which keeps the window in which visitors could see a mix of old and new files short. `--delete-after` removes files that no longer exist in the build only once the new files are in place. `--chmod` publishes everything with the permissions Caddy needs, whatever the local file modes are.

Images and fonts are not precompressed, because formats such as WebP, AVIF, JPEG and WOFF2 are already compressed. Recompressing them gains almost nothing.

## Maintenance

**Operating system updates.** Security updates from Debian install automatically every day. Check the log at `/var/log/unattended-upgrades/` occasionally. Kernel updates need a reboot to take effect, so either reboot during a quiet period after kernel updates, or set `Unattended-Upgrade::Automatic-Reboot "true";` with a reboot time in a local file under `/etc/apt/apt.conf.d/`.

**Caddy updates.** Caddy comes from a third-party repository, so unattended-upgrades does not update it by default. Update it deliberately with `sudo apt update && sudo apt install --only-upgrade caddy`, then confirm with `caddy version` and `systemctl status caddy`. If you prefer automatic Caddy updates, find the repository's origin with `apt-cache policy caddy` and add it to `Unattended-Upgrade::Origins-Pattern` in a local file.

**Certificates.** Caddy renews certificates on its own. Nothing needs to be scheduled.

**Adding administrators or deploy keys.** Add keys to the cloud-init file first, then to the running server, so that a rebuild reproduces the change. Each additional deploy key should carry the same `command="rrsync -wo /srv/sites",restrict` prefix.

## Rebuild and recovery

To replace the server, whether after a failure, a compromise, a move to another region or a move to another provider:

1. Create a new Droplet with the same `cloud-config.yaml` and the `web-static` tag. It joins the Cloud Firewall automatically.
2. Verify the first boot as described above.
3. Copy the per-site files from the infrastructure repository into `/etc/caddy/sites/`, then validate and reload Caddy.
4. Run `deploy.sh` for each site against the new server.
5. Point each domain's A and AAAA records at the new addresses. Caddy obtains new certificates as each domain starts resolving to it.
6. Destroy the old Droplet once traffic has moved.

With low DNS TTLs set ahead of time, the switch takes minutes. If you want to skip certificate reissuance, copy `/var/lib/caddy/` from the old server before destroying it. This is optional, since reissuing a handful of certificates during a rebuild is routine.

For a move away from DigitalOcean, only two things change: the Cloud Firewall rules must be recreated in the new provider's equivalent, and the `doctl` commands are replaced by the new provider's tooling. The cloud-init file, the nftables ruleset, the Caddy configuration and the deploy script stay the same.

## Verification checklist

Run these checks after the first deploy of each site.

```shell
curl -sI https://example.org/ | head -n 1
curl -sI http://example.org/ | grep -i '^location'
curl -sI https://www.example.org/ | grep -i '^location'
curl -sI -H 'Accept-Encoding: zstd' https://example.org/ | grep -i '^content-encoding'
curl -sI -H 'Accept-Encoding: br' https://example.org/ | grep -i '^content-encoding'
curl -sI https://example.org/ | grep -i '^alt-svc'
curl -sI https://example.org/ | grep -i '^strict-transport-security'
curl -sI https://example.org/does-not-exist | head -n 1
```

The expected results are: a `200` response over HTTPS; HTTP redirecting to HTTPS; `www` redirecting to the apex; `content-encoding: zstd` and `content-encoding: br` for the respective requests, which confirms the precompressed sidecars are being served; an `alt-svc` header advertising `h3`, which confirms HTTP/3 is offered; an HSTS header; and a `404` for a missing page, served from the site's `404.html`.

## Corrections to the DigitalOcean guide

DigitalOcean's guide is a reasonable general baseline, but several of its steps are wrong or weaker than described, and several do not suit a static host.

- **The startup script has a syntax error.** Its last line, `if sshd -t -q; then systemctl restart sshd fi`, is missing a `;` before `fi`. Bash treats `fi` as an argument to `systemctl`, the `if` never closes, and the script fails at that point. Bash has already run the earlier lines, so the user is created and the `sed` edit is made, but SSH is never restarted, and the change only takes effect after the next reboot.
- **It is not cloud-config.** The guide calls its script a "cloud-config script", but it is a bash script. This setup uses declarative `#cloud-config` YAML, which is portable across providers and Incus.
- **Root login is not disabled.** `PermitRootLogin prohibit-password` still allows root to log in with a key. This setup uses `PermitRootLogin no`.
- **Password authentication is not explicitly disabled.** The script never sets `PasswordAuthentication`, so its claim to disable password login relies on image defaults. This setup sets it in a drop-in and through cloud-init.
- **The SSH edit can be overridden.** Editing `/etc/ssh/sshd_config` with `sed` can be overridden by files in `sshd_config.d`, which are read first, because the first value obtained wins. This setup writes a drop-in that sorts first.
- **The firewall serves no websites.** The guide's firewall allows only SSH inbound, and its `doctl` example opens SSH to IPv4 only. This setup allows TCP 80 and 443 and UDP 443 on both address families, and restricts SSH by source address.
- **The Droplet is oversized.** `s-2vcpu-2gb` is far more than static serving needs. This setup starts at 1 GB.
- **Backups protect the wrong thing.** For a static host, rebuilding from version-controlled configuration is faster, cheaper and more portable than restoring a provider disk image.
- **There is no host firewall.** The guide relies entirely on the provider's firewall. This setup adds nftables so the policy moves with the server.

## References

<a id="ref-caddy-nd-a"></a>Caddy. (n.d.-a). *file_server*. Caddy documentation. Retrieved October 24, 2026, from https://caddyserver.com/docs/caddyfile/directives/file_server
[Return to citation](#cite-caddy-nd-a)

<a id="ref-caddy-nd-b"></a>Caddy. (n.d.-b). *import*. Caddy documentation. Retrieved October 24, 2026, from https://caddyserver.com/docs/caddyfile/directives/import
[Return to citation](#cite-caddy-nd-b)

<a id="ref-caddy-nd-c"></a>Caddy. (n.d.-c). *Install*. Caddy documentation. Retrieved October 24, 2026, from https://caddyserver.com/docs/install
[Return to citation](#cite-caddy-nd-c)

<a id="ref-caddy-docker-nd"></a>Caddy Docker Maintainers. (n.d.). *caddy* [Docker official image documentation]. Docker Hub. Retrieved October 24, 2026, from https://hub.docker.com/_/caddy
[Return to citation](#cite-caddy-docker-nd)

<a id="ref-cloud-init-nd"></a>cloud-init. (n.d.). *All cloud config examples*. cloud-init documentation. Retrieved October 24, 2026, from https://docs.cloud-init.io/en/latest/reference/examples.html
[Return to citation](#cite-cloud-init-nd)

<a id="ref-debian-2025"></a>Debian. (2025, August 9). *Debian 13 "trixie" released* [Press release]. https://www.debian.org/News/2025/20250809
[Return to citation](#cite-debian-2025)

<a id="ref-debian-wiki-nd"></a>Debian Wiki. (n.d.). *nftables*. Retrieved October 24, 2026, from https://wiki.debian.org/nftables
[Return to citation](#cite-debian-wiki-nd)

<a id="ref-digitalocean-nd-a"></a>DigitalOcean. (n.d.-a). *doctl compute firewall*. DigitalOcean documentation. Retrieved October 24, 2026, from https://docs.digitalocean.com/reference/doctl/reference/compute/firewall/
[Return to citation](#cite-digitalocean-nd-a)

<a id="ref-digitalocean-nd-b"></a>DigitalOcean. (n.d.-b). *Linux images for Droplets*. DigitalOcean documentation. Retrieved October 24, 2026, from https://docs.digitalocean.com/products/droplets/details/images/
[Return to citation](#cite-digitalocean-nd-b)

<a id="ref-digitalocean-2026a"></a>DigitalOcean. (2026a, January 7). *Firewalls limits*. DigitalOcean documentation. https://docs.digitalocean.com/products/networking/firewalls/details/limits/
[Return to citation](#cite-digitalocean-2026a)

<a id="ref-digitalocean-2026b"></a>DigitalOcean. (2026b, August 7). *Set up a production-ready Droplet*. DigitalOcean documentation. https://docs.digitalocean.com/products/droplets/getting-started/recommended-droplet-setup/
[Return to citation](#cite-digitalocean-2026b)

<a id="ref-geniar-2026"></a>Geniar, M. (2026). *Running HTTP/3 with Caddy in 2026: It just works*. ma.ttias.be. https://ma.ttias.be/running-http3-with-caddy-2026/
[Return to citation](#cite-geniar-2026)

<a id="ref-linuxcapable-nd"></a>LinuxCapable. (n.d.). *How to configure unattended upgrades on Debian 13, 12 and 11*. Retrieved October 24, 2026, from https://www.linuxcapable.com/how-to-enable-configure-unattended-upgrades-on-debian-11/
[Return to citation](#cite-linuxcapable-nd)

<a id="ref-openssh-nd"></a>OpenSSH. (n.d.). *sshd_config(5): OpenSSH daemon configuration file* [Manual page]. Retrieved October 24, 2026, from https://man.cx/sshd_config(5)
[Return to citation](#cite-openssh-nd)

<a id="ref-rsync-2025"></a>The rsync project. (2025, January 15). *rrsync(1): A script to setup restricted rsync users via ssh logins* [Manual page]. https://man7.org/linux/man-pages/man1/rrsync.1.html
[Return to citation](#cite-rsync-2025)
