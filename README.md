# omsa8 — Dell OpenManage Server Administrator 8.4 for Arch Linux

AUR packaging of **Dell OpenManage Server Administrator (OMSA) 8.4** so it runs on
modern Arch Linux. OMSA 8.4 is the branch that supports **11th–13th generation
PowerEdge** servers (e.g. PowerEdge T320/R420/R720). It provides:

- `omreport` / `omconfig` CLI for hardware, RAID/PERC, sensors, and the system log
- an optional web GUI on `https://<host>:1311`

## Packages

| Package | Contents |
|---------|----------|
| `omsa8` | Data engine + `omreport`/`omconfig` CLI (the apt `srvadmin-base` set) |
| `omsa8-webserver` | Web GUI on port 1311 (Tomcat + bundled JRE); depends on `omsa8` |

## How it works

OMSA 8.4 is proprietary software built for Ubuntu/RHEL of its era. These packages
**repackage Dell's official `.deb`s** (downloaded from `linux.dell.com`) and pair them
with a small set of **2016-era compatibility libraries** (libxml2 2.9.4, libxslt
1.1.28, ICU 57, libsmbios 2.2, ncurses 5) extracted from Ubuntu's archive.

Those old libraries are **isolated** under `/opt/dell/srvadmin/omsa-compat` and are fed
to OMSA only through `LD_LIBRARY_PATH` — they are **never** added to the system linker
cache, so your host's own `libxml2`/`libxslt`/etc. are left completely untouched.

## Install

```sh
# with an AUR helper
yay -S omsa8                 # CLI / instrumentation only
yay -S omsa8-webserver       # add the web GUI (pulls in omsa8)
```

The build downloads ~250 MB of Dell + Ubuntu packages.

## Usage

```sh
# start the data engine
sudo systemctl enable --now dell-openmanage.service

# query hardware (use the full path; sudo's secure PATH excludes /opt)
sudo /opt/dell/srvadmin/bin/omreport storage controller
sudo /opt/dell/srvadmin/bin/omreport storage pdisk controller=0
sudo /opt/dell/srvadmin/bin/omreport chassis
```

### Web GUI

```sh
sudo systemctl enable --now dell-openmanage-web.service
```

Then browse to `https://<this-host>:1311` (self-signed certificate — accept the
warning). **Log in with your own Linux user and password** (e.g. your `sudo`
account) — not `root`, unless `root` actually has a password set.

Roles (Administrator / Power User / User) are mapped in
`/opt/dell/srvadmin/etc/omarolemap`. By default `root`, the `mike` user, and the
`root`/`wheel` groups are mapped to Administrator — edit this file for your site.

#### Behind a reverse proxy

The web GUI works behind a reverse proxy (e.g. an Nginx vhost or a Cloudflare
tunnel) — point the proxy at `https://<this-host>:1311` and allow self-signed
origins. The port-1311 Tomcat connector is configured with a 64 KB maximum HTTP
header size so that the headers and cookies a proxy adds (Cloudflare's `CF-*`,
`X-Forwarded-*`, and `__cf_bm` cookie, plus a long `Referer`) do not overflow
Tomcat's 8 KB default. Without that headroom, heavier requests return `400` and
the GUI bounces back to the login screen in a redirect loop.

## Requirements

- `x86_64`
- A Dell PowerEdge supported by OMSA 8.4 (11G–13G)
- The mainline kernel's `dcdbas` and IPMI modules (standard on the Arch `linux`
  kernel; loaded automatically)

## Licensing

OMSA itself is **proprietary Dell software**. This repository contains *packaging
only* — the PKGBUILD downloads the binaries from Dell's official repository and does
not redistribute them. The bundled compatibility libraries are open source (LGPL/MIT)
and are fetched from Ubuntu's archive. Use of OMSA is subject to Dell's license.

## Maintaining

`scripts/validate-package.sh` runs the same checks as CI (`.SRCINFO` diff,
`makepkg --verifysource`, `namcap`); pass `--build` for a full build.
`scripts/publish-aur-tree.sh` mirrors `PKGBUILD`, `.SRCINFO`, the `.install` files,
and the local source files to the AUR. The version is pinned to **8.4.0** (the last
OMSA release supporting 12G hardware), so there is no automatic upstream bump.
