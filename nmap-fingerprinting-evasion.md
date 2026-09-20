# Nmap Fingerprinting and Evasion Techniques

**Track:** eJPT prep — Host/Network Pentesting module
**Target:** Metasploitable2 — 192.168.56.101 (Ubuntu 8.04, kernel 2.6.24-16-server, i686)

## Objective

Building on earlier basic Nmap scan types (Day 16), this piece goes deeper
into service/version detection, OS fingerprinting, NSE scripting, and the
classic evasion techniques (fragmentation, decoys) that round out Nmap's
core exam-relevant feature set.

## OS fingerprinting (`-O`)

```bash
nmap -O 192.168.56.101
```

Nmap infers the target OS from subtle TCP/IP stack behavior — TTL defaults,
window size, TCP option ordering — and reports a confidence percentage
rather than a definitive answer. A direct comparison against a second,
more hardened target (e.g. a home router with most ports filtered) would
be expected to show a meaningfully lower confidence score, since a
sparser, filtered attack surface gives Nmap fewer stack behaviors to
sample from. That comparison is still pending — postponed until the
comparison target is available again.

## NSE (Nmap Scripting Engine)

NSE is Nmap's built-in scripting engine — not a separate tool — running
`.nse` scripts organized into categories. The `nmap.org/nsedoc/` page is
the browsable reference for the full script catalog.

```bash
nmap --script=vuln 192.168.56.101        # detection-only vulnerability checks
nmap --script=auth 192.168.56.101        # default/weak auth checks
nmap --script=discovery 192.168.56.101   # broader service enumeration
nmap -sC 192.168.56.101                  # shorthand for the "default" script set
```

`--script=vuln` correctly flagged CVEs already confirmed and exploited
manually earlier in this curriculum (vsftpd, Samba, UnrealIRCd) —
a useful tool-vs-manual-method cross-check: automated scanning found the
same issues hands-on enumeration had already surfaced, at a fraction of
the time cost.

`--script=exploit` exists as a category too but was **not** run here — unlike
`vuln` (detection-only), scripts in this category can actually attempt
exploitation, which is a meaningfully different risk profile worth treating
with more caution even in an isolated lab.

## Fragmentation (`-f`)

```bash
nmap -f 192.168.56.101       # fragments packets into 8-byte chunks
nmap -ff 192.168.56.101      # smaller fragments (16-byte MTU)
nmap --mtu 24 192.168.56.101 # custom fragment size, must be a multiple of 8
```

Splits probe packets into small IP fragments so that firewalls/IDS
inspecting one packet at a time — without reassembling fragments first —
may fail to recognize the traffic as a scan. Largely historical at this
point: modern inspection systems reassemble fragments before pattern
matching, so this is exam-relevant theory more than a technique with
practical bite against current defenses. No observable difference in scan
results against this lab's target, since nothing here performs
fragment-level inspection.

## Decoys (`-D`)

```bash
nmap -D decoy1,decoy2,decoy3,ME 192.168.56.101
nmap -D RND:5 192.168.56.101
```

Correct syntax is a comma-separated list of decoy source addresses, not a
single IP. The optional `ME` keyword places the real scanning IP at a
specific position in that list rather than leaving its position to Nmap.
`RND:N` generates N random decoy addresses instead of requiring real ones
to be supplied manually.

**Mechanism, precisely:** this isn't real-time confusion for a live
analyst — it's about polluting the **target's logs** with multiple
apparent source IPs, so that a defender reviewing logs after the fact
can't trivially isolate which source actually performed the scan without
further correlation. Decoy IPs also need to be live, reachable hosts —
dead or unreachable decoys can make a scan stand out as obviously
artificial (e.g. failing to complete a TCP handshake) rather than blending
in.

## Takeaways

- **OS fingerprint confidence is itself informative**, separate from the
  OS guess — a lower-confidence result against a hardened target is a
  useful signal on its own, worth comparing across targets with visibly
  different attack surfaces.
- **NSE's `vuln` category is a strong sanity-check tool**, not a
  replacement for manual verification — it corroborated findings already
  confirmed by hand, which is the ideal relationship between automated
  scanning and manual technique.
- **Fragmentation and decoys are largely exam-theory today** — both were
  meaningfully more effective against the network inspection tooling
  common when they were introduced than against typical modern defenses,
  but remain standard knowledge for eJPT and worth understanding precisely
  (exact syntax, actual mechanism) rather than approximately.

## Next steps

- OS-fingerprint confidence comparison against a second, more hardened
  target (pending availability)
- Move to web interface attack surface (Burp Suite / DVWA) as the next new
  technique area
