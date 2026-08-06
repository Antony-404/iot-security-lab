# Firmware Analysis: TP-Link TL-WR841N (OpenWrt Build)

## Objective

Extract and analyze router firmware to identify hardcoded credentials, weak
default configurations, and exposed services — the kind of static,
pre-deployment findings that never require network access to the live
device, only a copy of its firmware update file.

## Target Selection: A Pivot Worth Documenting

The original target was TP-Link's stock TPOS firmware for the TL-WR841N
(`TL-WR841N-14.20-TPOS-up-noboot-256B_2025-11-14_18.09.13.bin`), downloaded
in Week 1 alongside initial CVE research on this device.

Signature scanning with Binwalk found no recognizable filesystem
(squashfs, JFFS2, or otherwise) in this image — only a single genuine LZMA
block (identified via MIPS opcode inspection as the kernel), surrounded by
dozens of false-positive signature matches from binwalk misreading the
middle of one large compressed region as separate file starts.

Research confirmed this wasn't a tooling gap: TP-Link's TPOS format uses a
proprietary filesystem (MiniFS) that has only recently been reverse-engineered
by academic researchers (2024), specifically because of the absence of public
documentation. Existing public extraction workflows exist for older,
OpenWrt-compatible TP-Link firmware, not TPOS.

**Decision:** rather than replicate months of dedicated reverse-engineering
research, the target was switched to an official OpenWrt build for the same
device family — `openwrt-18.06.8-ar71xx-tiny-tl-wr841-v8-squashfs-factory.bin`.
Same device line (keeping Week 1's CVE research relevant), but a
well-documented, squashfs-based format that tooling actually supports. The
TPOS dead-end is included here because a genuine "hit an undocumented
proprietary format and pivoted" is realistic security work, not a failure to
hide.

## Environment

- WSL1 (Ubuntu) — WSL2 was abandoned for this task after repeated
  NAT/vEthernet networking failures blocked `apt` entirely (same root cause
  as an earlier Mosquitto/WSL2 networking issue). WSL1 shares the Windows
  host's network stack directly, sidestepping the problem.
- Binwalk v2.4.3, `squashfs-tools` (for `unsquashfs`)

## Signature Scan

```
binwalk openwrt-18.06.8-ar71xx-tiny-tl-wr841-v8-squashfs-factory.bin
```

```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
512           0x200           LZMA compressed data, ... uncompressed size: 4297604 bytes
675247        0xA4DAF         JBOOT STAG header, image id: 1, ...
1366208       0x14D8C0        Squashfs filesystem, little endian, version 4.0,
                               compression:xz, size: 2283546 bytes, 1138 inodes,
                               blocksize: 262144 bytes, created: 2020-02-27 21:25:59
```

Unlike the TPOS scan, every entry here is a genuine, well-formed match: the
kernel (LZMA), a bootloader partition header (JBOOT STAG), and a real
Squashfs filesystem — no false-positive noise.

## Extraction

```
sudo binwalk -e --run-as=root openwrt-18.06.8-ar71xx-tiny-tl-wr841-v8-squashfs-factory.bin
```

`--run-as=root` (and running binwalk itself under `sudo`) was required
because squashfs extraction creates special file types — device nodes and
symlinks — that need elevated permission to write, even inside a normal
home directory.

Binwalk sanitized several symlinks that pointed outside the extraction
directory (e.g. `etc/resolv.conf -> /tmp/resolv.conf`), rewriting them to
`/dev/null` as a safety measure. This is expected, intentional behavior, not
an extraction failure.

Result: a complete `squashfs-root/` directory — a real router root
filesystem (`bin/`, `etc/`, `usr/`, `www/`, etc.).

## Findings

### 1. Root account has no password set

`etc/shadow`:
```
root::0:0:99999:7:::
```

The password hash field for `root` is empty. An empty hash in `/etc/shadow`
means the account can be authenticated with **no password at all**.

### 2. SSH permits root password login by default

`etc/config/dropbear`:
```
config dropbear
    option PasswordAuth      'on'
    option RootPasswordAuth  'on'
    option Port              '22'
    option BannerFile        '/etc/banner'
```

SSH (via Dropbear) is configured to accept password-based root login on
port 22.

### Combined impact

Findings 1 and 2 together mean: **any device running this firmware, left
unconfigured after a fresh flash, is reachable over SSH with unauthenticated
root access.** Neither finding alone is dramatic — a blank shadow field is
just a file, and password SSH is normal — but together they describe a real,
remotely reachable, zero-credential entry point.

### Important caveat

This is **known, intentional OpenWrt behavior**, not an overlooked flaw.
OpenWrt ships this way deliberately, expecting the user to set a root
password (via the web UI or `passwd`) during first-boot setup. It becomes a
genuine vulnerability only when that setup step is skipped — which happens
regularly in real deployments, and is exactly the class of exposure that
passive recon (e.g. Shodan scans of freshly-flashed or misconfigured
devices) surfaces at scale.

### 3. Web admin interface (LuCI)

`www/index.html` redirects to `/cgi-bin/luci/`. The CGI entry point is a
minimal 135-byte launcher:

```lua
#!/usr/bin/lua
require "luci.cacheloader"
require "luci.sgi.cgi"
luci.dispatcher.indexcache = "/tmp/luci-indexcache"
luci.sgi.cgi.run()
```

This hands off to LuCI's compiled Lua dispatcher elsewhere in the
filesystem. Auditing LuCI's internal routing/auth logic was out of scope for
this exercise (a much larger undertaking — real source auditing, not
firmware extraction) but is noted as a natural next step if this analysis
were extended.

## Summary

| Item | Result |
|---|---|
| Original target (TPOS) | Proprietary, undocumented format — no usable filesystem extracted |
| Pivot target (OpenWrt) | Clean squashfs extraction, full filesystem recovered |
| Root password | Not set (empty hash in `/etc/shadow`) |
| SSH root login | Enabled by default (`RootPasswordAuth 'on'`) |
| Combined risk | Unauthenticated remote root access if device setup is skipped |
| Classification | Default-insecure by design, not a coding flaw |

## Tools

Binwalk v2.4.3, squashfs-tools, WSL1 (Ubuntu)
