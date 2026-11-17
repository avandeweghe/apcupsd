# apcupsd with outlet-group support for PowerChute (pcnet)

This repository is Debian's `apcupsd` packaging
([salsa.debian.org/debian/apcupsd](https://salsa.debian.org/debian/apcupsd))
plus a few patches that make the **pcnet** driver (PowerChute Network Shutdown
protocol) work properly with UPSes that have **switched outlet groups**, such as
the Smart-UPS X (SMX) and Smart-UPS (SMT) lines with a network management card.

The work lives on the `pcnet-outlet-group` branch, based on Debian's
`3.14.14-5` (the version in Debian 13 "trixie"). The changes are ordinary
Debian quilt patches in `debian/patches/`, so the tree builds into normal
`.deb` packages.

## Why

On a UPS with outlet groups, the management card assigns each PowerChute client
to a group. When a group's shutdown starts (for example, load shedding after N
seconds on battery), the card tells that group's clients by setting `SDn=1`
(`SD1`, `SD2`, `SD3`, ...) in its status broadcast, then turns the group off
after the group's Power Off Delay.

Stock apcupsd only understands the whole-UPS flag (`SD`), so a server on a
load-shed group never shut down: the card cut its power without warning.

## Features

### 1. Follow an outlet group's shutdown (patch 17)

`DEVICE` accepts an optional outlet group after the port:

```
UPSCABLE ether
UPSTYPE  pcnet
DEVICE   ipaddr:username:passphrase[:port[:group]]
```

Leave `port` empty to keep the default (3052). For example, to follow group 3:

```
DEVICE 10.0.0.5:apc:your-auth-phrase::3
```

With a group set, `SD<group>=1` from the card triggers the same clean remote
shutdown as a whole-UPS `SD=1`:

```
Management card started shutdown of outlet group 3
Remote shutdown requested.
Initiating system shutdown!
```

- `SD` and `SDn` are combined, so the order of keys in the packet doesn't matter.
- Flags for other groups are ignored.
- Without a group, behaviour is exactly the same as stock apcupsd.

### 2. Outlet group status in `apcaccess` (patch 19)

When a group is configured, the status output (and the NIS server on port 3551,
which Home Assistant and other tools read) includes:

```
OUTLGRP  : 3
OUTLNAME : Outlet Group 3
OUTLSTAT : ON
```

| Field | Meaning |
|---|---|
| `OUTLGRP` | Group number from `DEVICE` |
| `OUTLNAME` | Group name as configured on the card (`SNn`) |
| `OUTLSTAT` | `ON`, `SHUTTING DOWN` (card started the group's shutdown, outlets still on), `OFF`, or `UNKNOWN` (no data yet since start-up) |

### 3. Version marker (patch 19)

The release string ends in `+pcnetgroupN`, so the patched build is easy to tell
apart from stock in `apcaccess`, logs and Home Assistant:

```
VERSION  : 3.14.14+pcnetgroup2 (31 May 2016) debian
```

The package version follows the same scheme (`3.14.14-5+pcnetgroup2`).

### 4. Manual builds with current docutils (patch 18)

`doc/manual/maintaining_ups.rst` contained a Latin-1 character that Debian 13's
docutils rejects, which failed the package build. It's converted to UTF-8.

## Killpower with a shared UPS: read this

After apcupsd shuts a machine down, Debian's shutdown hooks run
`apccontrol killpower`. Over pcnet that tells the card to shut down the
**whole UPS**, which turns off every outlet group, not just the one the
machine is on.

On a UPS shared by several machines, disable that until group-aware killpower
lands. Edit the `killpower)` case in `/etc/apcupsd/apccontrol` (a conffile, so
package upgrades keep your change) to only log, for example:

```sh
    killpower)
	echo "apccontrol: killpower ignored on this host (shared UPS)" | logger -t apcupsd
    ;;
```

The card's own load shedding still turns the group off after its Power Off
Delay, so a group-following host doesn't need killpower in normal use.

**Planned (patch 20):** in group mode, killpower turns off only the configured
outlet group (SNMP outlet-group "delayed off"), and never the whole UPS.

## Card settings that go with this

On the management card, for the outlet group the server is plugged into
(Configuration → Outlet Groups):

- **Turn off when power failure lasts longer than**: when the group's shutdown
  starts; the server begins shutting down at this point.
- **Skip outlet off delay**: unchecked, so the card waits before cutting power.
- **Power Off Delay**: longer than the server's real shutdown time, with margin.
  Measure it; a Proxmox host migrating VMs can take much longer than a plain
  shutdown.
- **Stay off after power returns**: unchecked, so the group comes back on.
- Set the server's BIOS to power on when AC is restored.

## Building

GitHub Actions (`.github/workflows/build-deb.yml`) builds the packages in a
`debian:trixie` container on every push to `pcnet-outlet-group`, checks the new
code is in the binary, and uploads the `.deb` files as the `apcupsd-debs`
artifact.

To build locally on Debian 13:

```sh
sudo apt-get install build-essential devscripts equivs git
git clone -b pcnet-outlet-group https://github.com/avandeweghe/apcupsd.git
cd apcupsd
sudo mk-build-deps -i -r debian/control
git archive --prefix=apcupsd-3.14.14/ upstream/3.14.14 | gzip -n > ../apcupsd_3.14.14.orig.tar.gz
dpkg-buildpackage -b -us -uc
```

## Installing

```sh
sudo apt-get install -o Dpkg::Options::=--force-confold \
  ./apcupsd_3.14.14-5+pcnetgroup2_amd64.deb ./apcupsd-doc_3.14.14-5+pcnetgroup2_all.deb
sudo apt-mark hold apcupsd apcupsd-doc
```

`--force-confold` keeps your existing `/etc/apcupsd` files. The hold stops a
Debian update from replacing the patched build.

## Tested with

- Smart-UPS X 1500 (SMX1500RM2U) with a network management card, PowerChute
  status protocol `MV=v1.7.0`.
- Debian 13 / Proxmox VE 9 host following outlet group 3: the card's group-3
  load shed produced `SD3=1`, and the host shut down cleanly before the group
  was switched off.

## Patches

| Patch | What it does |
|---|---|
| `17-pcnet-outlet-group-shutdown.patch` | Optional `:group` in `DEVICE`; act on `SDn` |
| `18-manual-utf8.patch` | Convert a manual file to UTF-8 so docs build |
| `19-apcaccess-outlet-group.patch` | `OUTLGRP` / `OUTLNAME` / `OUTLSTAT`; `+pcnetgroupN` version marker |

These patches have not been sent upstream.
