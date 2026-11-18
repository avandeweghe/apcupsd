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

**From `+pcnetgroup3` on, this is handled for you when `DEVICE` names an
outlet group**: pcnet killpower is skipped and logged ("Killpower skipped:
following outlet group N ...") instead of shutting down the whole UPS
(patch 20).

Without a group, on a UPS shared by several machines, disable it yourself.
Edit the `killpower)` case in `/etc/apcupsd/apccontrol` (a conffile, so
package upgrades keep your change) to only log, for example:

```sh
    killpower)
	echo "apccontrol: killpower ignored on this host (shared UPS)" | logger -t apcupsd
    ;;
```

Killpower isn't needed with pcnet: the card's own load shedding turns each
group off after its Power Off Delay and back on when power returns.

Tested on an AP9630 (firmware 6.4.0): pcnet's killpower always starts a
**whole-UPS** graceful shutdown, even when the sending host is registered on a
specific outlet group in the card's PowerChute client list. The card log shows
all switched groups "commanded to turn off, sequenced", initiated by the
client's IP. The pcnet protocol has no group-scoped power-off. If you ever
need one, the documented route is SNMP: set `upsOutletGroupControlCommand`
(`.1.3.6.1.4.1.318.1.1.1.12.3.2.1.3.<group>`) to
`upsOutletGroupDelayedRebootShutdown` (10).

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
- **Reboot Duration**: the minimum time the group stays off before the Power
  On Delay starts. Once a group's shutdown has started, the card completes the
  off/on cycle even if mains returns early, so a halted server always gets a
  real power cycle. Time back on = Reboot Duration + Power On Delay.
- Set the server's BIOS to power on when AC is restored.
- The card does **not** wait for clients to finish shutting down: it cuts the
  group when the Power Off Delay expires.

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

## Installing with apt

CI publishes a signed APT repository for Debian 13 (trixie, amd64) at
<https://avandeweghe.github.io/apcupsd/>. Signing key fingerprint:
`8F6C 14E2 F7D8 2671 607E  F23F E4C8 3274 5D4B 6194` (also in `apcupsd-pcnet.asc`).

On each node, as root:

```sh
curl -fsSL https://avandeweghe.github.io/apcupsd/apcupsd-pcnet.gpg \
  -o /usr/share/keyrings/apcupsd-pcnet.gpg

echo "deb [signed-by=/usr/share/keyrings/apcupsd-pcnet.gpg] https://avandeweghe.github.io/apcupsd trixie main" \
  > /etc/apt/sources.list.d/apcupsd-pcnet.list

cat > /etc/apt/preferences.d/apcupsd-pcnet <<'EOF'
Package: apcupsd apcupsd-doc apcupsd-cgi
Pin: release o=apcupsd-pcnet
Pin-Priority: 1001
EOF

apt-get update
apt-get install -o Dpkg::Options::=--force-confold apcupsd
```

- The **pin** keeps this build even if Debian publishes a higher version, while
  still taking updates from this repo with a normal `apt upgrade`. Use it
  instead of `apt-mark hold` (remove any hold first: `apt-mark unhold apcupsd apcupsd-doc`).
- `--force-confold` keeps an existing `/etc/apcupsd` configuration.

Then configure the node in `/etc/apcupsd/apcupsd.conf` and restart apcupsd:

```
UPSCABLE ether
UPSTYPE  pcnet
DEVICE   <card-ip>:<user>:<auth-phrase>::<outlet-group>
```

and check with `apcaccess` (look for `OUTLGRP`, `OUTLSTAT` and `STATUS : ONLINE`).
On the card, register the node's IP in the PowerChute client list on the same
outlet group.

## Tested with

- Smart-UPS X 1500 (SMX1500RM2U) with a network management card, PowerChute
  status protocol `MV=v1.7.0`.
- Debian 13 / Proxmox VE 9 host following outlet group 3: the card's group-3
  load shed produced `SD3=1`, and the host shut down cleanly before the group
  was switched off. With mains restored mid-sequence, the card still completed
  the group's off/on cycle and the host booted on its own.

## Patches

| Patch | What it does |
|---|---|
| `17-pcnet-outlet-group-shutdown.patch` | Optional `:group` in `DEVICE`; act on `SDn` |
| `18-manual-utf8.patch` | Convert a manual file to UTF-8 so docs build |
| `19-apcaccess-outlet-group.patch` | `OUTLGRP` / `OUTLNAME` / `OUTLSTAT`; `+pcnetgroupN` version marker |
| `20-pcnet-group-skip-killpower.patch` | In group mode, skip pcnet killpower (it would shut down the whole UPS) |

These patches have not been sent upstream.
