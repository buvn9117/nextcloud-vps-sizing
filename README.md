# Nextcloud file server setup: how much RAM a self-hosted cloud really needs, which install path to pick, and the VPS plans that fit ($12–$89/month)

Nextcloud is free software. The bill is the machine under it, and the sizing question has a published answer: 128MB of RAM is the absolute floor, 512MB is what the project recommends for a basic single-user setup, and a full Nextcloud Hub install with several users wants 3GB to 8GB. The rest of this page turns those numbers into a server choice, then covers the three ways to install Nextcloud and the default limits that decide whether the thing is still pleasant to use a month after setup.

> **Quick pick:** a household or small team running the official All-in-One install with online Office and image previews fits the 8GB / 6 vCPU plan at $39/month, which sits in the middle of Nextcloud's own 3–8GB range. One or two people syncing files and photos can start on the 2GB plan at $12/month. 👉 [See the current SpinServers KVM VPS plans and prices](https://bit.ly/Spinservers).

## SpinServers VPS plans, side by side

| Plan | CPU / RAM | Storage | Monthly transfer | Price (USD) | Order |
| --- | --- | --- | --- | --- | --- |
| KVM 2vCPU 2GB | 2 vCPU / 2 GB DDR4 | 40 GB RAID-10 SSD | 4 TB | $12.00/mo | [Order](https://bit.ly/Spinservers) |
| KVM 4vCPU 4GB | 4 vCPU / 4 GB DDR4 | 80 GB RAID-10 SSD | 8 TB | $29.00/mo | [Order](https://bit.ly/Spinservers) |
| KVM 6vCPU 8GB | 6 vCPU / 8 GB DDR4 | 120 GB RAID-10 SSD | 8 TB | $39.00/mo | [Order](https://bit.ly/Spinservers) |
| KVM 8vCPU 8GB | 8 vCPU / 8 GB DDR4 | 160 GB RAID-10 SSD | 10 TB | $45.00/mo | [Order](https://bit.ly/Spinservers) |
| KVM 12vCPU 12GB | 12 vCPU / 12 GB DDR4 | 160 GB RAID-10 SSD | 10 TB | $62.00/mo | [Order](https://bit.ly/Spinservers) |
| KVM 16vCPU 16GB | 16 vCPU / 16 GB DDR4 | 320 GB RAID-10 SSD | 15 TB | $74.00/mo | [Order](https://bit.ly/Spinservers) |
| Hybrid KVM 32 vCPU / 32GB | 32 vCPU / 32 GB | 500 GB SSD | 20 TB | $89.00/mo | [Order](https://bit.ly/Spinservers) |

Every plan is KVM with a 1Gbps port and one IPv4 plus one IPv6 address. The prices are the "starting from" monthly rates listed on SpinServers' KVM VPS page, so treat them as the entry point for each configuration rather than a fixed quote.

One line on that page matters more for Nextcloud than for a typical web app: outbound port 25 is closed by default on all SpinServers VPS plans, and opening it takes a support ticket with a justification. Nextcloud sends mail for password resets, share notifications and Talk invites, so plan for a relay (covered below) or count on the ticket being approved.

SpinServers also sells dedicated servers. A single Nextcloud instance doesn't need one until storage or hardware transcoding outgrows 500GB of disk and 32GB of RAM.

## How much RAM does a Nextcloud file server actually need?

Three numbers come from Nextcloud's own FAQ, and they measure the software, not the box around it:

- **128MB** is the stated absolute minimum.
- **512MB** is the recommendation for "the most basic single-user setup".
- **3GB to 8GB** is what a full Nextcloud Hub installation with multiple users needs.

Those are vendor floors, not a benchmark of your photo library or your sync client count. Mapping them onto real plans:

| Who is using it | Nextcloud's guidance | Plan that fits |
| --- | --- | --- |
| 1–2 people, file and photo sync | 512MB for a basic single-user setup | KVM 2GB, $12 |
| Family or small team with Calendar, Talk, previews | low end of the 3GB range | KVM 4GB, $29 |
| Team Hub with Office co-editing | 3–8GB | KVM 8GB / 6 vCPU, $39 |
| Same, with a larger data disk | 3–8GB, storage-bound | KVM 8GB / 8 vCPU, $45 |
| Many accounts, heavy previews or video transcoding | top of the range and above | KVM 12GB or 16GB, $62–$74 |

A caveat on the smallest plan. The All-in-One install runs a set of containers around Nextcloud itself: PostgreSQL as the database, Redis for distributed caching and file locking, APCu as local cache, PHP-FPM with Opcache, Apache and Caddy. The 512MB figure describes Nextcloud, not that stack. Two gigabytes is a realistic floor for AIO with one or two users; 512MB is not, even though it is what the software itself will boot on. Running the plain Docker image with SQLite on the same box leaves more headroom, but you lose Redis-backed file locking and the built-in cache, which is the trade you are making.

The 512MB that shows up in AIO's documentation is a different number entirely: the default memory ceiling per PHP process. Don't add it to the 3–8GB range.

## Three ways to install it, and which one to pick

### All-in-One: the official method

AIO is Nextcloud's own installation method, driven from a single web interface for install, updates and maintenance. It arrives with PostgreSQL, Redis, APCu, PHP-FPM and Opcache already wired together, scores A+ on the Nextcloud security scan, ships ffmpeg, smbclient and nodejs, enables HTTP/2 and HTTP/3, turns on Pretty URLs (no `index.php` in links), and gets a Let's Encrypt certificate automatically. One domain covers everything.

It also handles the parts people usually get wrong. Daily backups can be switched on in the interface and run through Borg, with restore, integrity check and repair. An instance can be restored onto a new AIO setup from the archive plus its password. You can put AIO behind an existing reverse proxy, a Cloudflare Tunnel or Tailscale, and if you don't own a domain you can register a free deSEC subdomain (`*.dedyn.io`) from the interface, with DNS records maintained automatically. Migration into AIO from an older Nextcloud install is supported, and so is migrating back out to a VM-based setup.

### The official Docker image

Two tags cover most deployments. `apache` bundles Nextcloud with Apache and exposes port 80; it is what `latest` points to. `fpm` exposes port 9000 and expects a separate web server such as nginx in front of it, and the documentation is explicit that you should not publish the fpm port to the host. SQLite is the default database, with the setup wizard offering MySQL/MariaDB or PostgreSQL instead, and a database container can be linked in.

This path gives you control over every component, and it makes you responsible for all of them: Redis is a separate container, TLS is yours to terminate, and updating means pulling a new image.

### Source or distribution packages

The server repository is AGPLv3, contributions don't require a CLA, and copyright stays with the individual contributors. It is also the path with the sharpest warning attached: the `stable*` branches should never be used on production systems. A `master` checkout is missing default apps such as the first-run wizard and Activity, which have to be cloned into `apps` by hand, and third-party components are git submodules that need initialising first. Fine for development against PHPUnit, Behat, Vitest and Playwright. Wrong tool for a file server your family depends on.

The short version: if you don't want to run a stack, use AIO. If you already operate nginx and a database and want to keep owning them, use the Docker image. Source builds are for people writing patches.

## Four defaults that bite after install

**Upload limits come in three different numbers.** AIO allows up to 10GB per file on public links by default, adjustable, and logged-in users push much larger files through the web interface or sync clients because chunking kicks in. The Docker image sets `PHP_UPLOAD_LIMIT` and `PHP_MEMORY_LIMIT` to 512M each, with `APACHE_BODY_LIMIT` at 1073741824 bytes (1GiB) and `0` meaning unlimited. Those are defaults for different products, not a contradiction, but they explain why a 3GB file uploads fine through the desktop client and fails through a public link. AIO also sets PHP and web server timeouts to 3600 seconds for exactly this reason.

**Behind a proxy, set `TRUSTED_PROXIES`.** The Apache image rewrites the client address from `X-Real-IP` when a request arrives from a private range. To have Nextcloud read `X-Forwarded-Host`, `X-Forwarded-Proto` and `X-Forwarded-For` instead, you disable the IP rewrite and add the proxy's address to `TRUSTED_PROXIES`. Skip it and your logs fill with proxy IPs while Nextcloud builds links with the wrong scheme.

**Don't rename the AIO container or volume.** `nextcloud-aio-mastercontainer` and `nextcloud_aio_mastercontainer` are hard dependencies of the built-in backup. Rename either and the backup stops working, quietly.

**Upgrades overwrite `/var/www/html`.** Anything you put in that directory is removed during installation and upgrades unless it is listed in `upgrade.exclude`. Custom volumes belong outside `/var/www/html`, ideally mounted read-only.

## Mail: port 25 is closed, so use a relay

The Docker image won't send a single message until at least `SMTP_HOST`, `MAIL_FROM_ADDRESS` and `MAIL_DOMAIN` are set. Point `SMTP_HOST` at a relay over 587 (STARTTLS) or 465 (SSL) and the SpinServers port 25 block never comes into play, since the restriction is specific to port 25. The same applies to AIO's mail settings.

AIO does offer an optional mail server container if you want Nextcloud to send directly. That one needs port 25, so it needs the ticket. For most setups a relay is less work than running an MTA and keeping it off blocklists.

## Backups decide how much disk you buy

AIO's daily Borg backup writes a repository you can keep locally or push to a remote Borg target, and it can also cover external Docker volumes or host paths. A full instance restore needs only the archive and the password.

The disk arithmetic is the part people skip. A local repository on the same disk means your data effectively exists twice, so a 30GB data directory needs roughly 60GB before you have room to grow. That's the difference between the 40GB plan and the 120GB or 160GB ones. The alternative is keeping the Borg repository remote and staying on a smaller disk. Decide which before you order, because Nextcloud data directories only get bigger.

## Who should skip self-hosting Nextcloud

Nextcloud is software, not a storage appliance. A NAS is a device on your network that stores and serves files; Nextcloud gives you file storage and sync plus calendar, chat and video calls, and it can run on a NAS if you want both. If all you need is a folder on the LAN that a few devices can reach, a NAS involves less maintenance.

If you don't want to deal with Docker, DNS and certificates, Nextcloud's own site lists the alternatives: a hosting provider, a preinstalled device, a managed service, or the Enterprise edition from Nextcloud GmbH. The trade is straightforward. Self-hosting gives you the data and the configuration; a managed plan gives you your evenings back.

## Picking a size, and two checks before you order

Match the plan to the install method and the headcount:

- **1–2 people syncing files and photos, plain Docker image:** KVM 2GB at $12/month.
- **1–2 people on All-in-One:** step up to KVM 4GB at $29/month for the container stack and thumbnail generation.
- **Family or small team with Calendar, Talk and previews:** KVM 4GB, or the 8GB / 6 vCPU at $39 if you want the middle of Nextcloud's stated range.
- **Team Hub with Office co-editing:** KVM 8GB / 6 vCPU at $39, or the 8 vCPU version at $45 when the data disk matters more than cores.
- **Many accounts, heavy previews, hardware transcoding:** KVM 12GB at $62 or 16GB at $74.

Two things to confirm before checkout. First, RAM against install method: 512MB boots Nextcloud but will not carry AIO's PostgreSQL, Redis and PHP-FPM containers for real users. Second, disk against backups: a local Borg repository doubles your storage requirement, and a remote repository needs somewhere to live.

One purchasing note. SpinServers does not refund month-to-month service except for SLA credits on a cancelled service, though it says it may refund at its discretion, with payment reversals taking up to 48 hours. Size the plan correctly the first time rather than planning to swap it later.

👉 [Check current SpinServers VPS pricing and pick the plan that matches your user count](https://bit.ly/Spinservers).
