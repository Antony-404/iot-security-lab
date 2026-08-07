# Shodan OSINT: MQTT Brokers in Kenya (port 1883)

**Date:** Day 13 (backfill session)
**Objective:** Passive reconnaissance of internet-facing MQTT brokers in Kenya using Shodan, to demonstrate the real-world impact of the MQTT auth-bypass attack practiced in the Day 10 lab, and to complete the last item of the Phase 1–2 attack-side backfill.

---

## Methodology

- Tool: [Shodan](https://www.shodan.io/), free-tier account
- Query: `port:1883 country:"KE"`
- All data used in this writeup was retrieved passively from Shodan's pre-existing index. Shodan continuously crawls the internet and grabs service banners in the background; running a search queries that stored index, it does not trigger a live scan or any connection from my machine to the target hosts.
- Free-tier accounts are capped at 2 pages of results, so this writeup covers a representative sample rather than the full result set.
- **No connections were made to any of the live hosts discussed below.** All observations are limited to what Shodan's banner data already shows. This boundary is intentional and is discussed further in the Ethics section.

---

## Scope: The Search

`port:1883 country:"KE"` returned **316 results** at time of search, concentrated heavily in Nairobi (304 of 316), with smaller counts in Mombasa, Eldoret, Kakamega, and Kisii. The top organizations were Metaverse Limited and Safaricom Limited — consistent with these being consumer/business ISP-assigned IP ranges rather than a single company's infrastructure.

---

## Finding 1 — Port Open ≠ MQTT Present

Not every host with port 1883 open is actually running an MQTT broker. Three of the first results returned banners that were clearly **HTTP**, not MQTT:

| IP | Banner |
|---|---|
| 138.113.42.133 | No data returned (port open, no banner captured) |
| 41.139.211.105 | Full IIS/Windows HTTP response (`Server: Microsoft-IIS/10.0`, ASP.NET) |
| 146.103.74.88 | nginx `400 Bad Request` |

**Takeaway:** these are most likely misconfigured port-forwarding rules on consumer routers/CPE, or multi-service hosts where something unrelated to MQTT happens to be bound to 1883. This is a useful reminder for real recon work — a port number is a hint, not a confirmation. Verifying the actual protocol response is a required step before treating a host as an MQTT target.

---

## Finding 2 — Open, Misconfigured Broker

**Host:** `102.68.86.115` (Web4Africa, Kenya — Nairobi)
**MQTT Connection Code: 0** (Connection Accepted)

Shodan's MQTT probe connected to this broker with no credentials supplied and was accepted outright. The broker then exposed its `$SYS` system topics, which include:

```
$SYS/broker/version
$SYS/broker/timestamp
$SYS/broker/uptime
$SYS/broker/clients/total
$SYS/broker/clients/maximum
$SYS/broker/clients/inactive
$SYS/broker/clients/disconnected
$SYS/broker/clients/active
$SYS/broker/clients/connected
$SYS/broker/clients/expired
```

**Why this matters:** `$SYS` topics are broker-internal telemetry that most MQTT brokers (Mosquitto included) publish by default and allow any connected client to subscribe to, unless explicitly restricted. On a broker with no authentication, this means anyone — not just Shodan — can passively pull:

- **Broker version** → can be cross-referenced against known CVEs for that specific release
- **Client counts (total/active/connected)** → gives an attacker a sense of scale: is this one test device, or a broker serving a real fleet?
- **Uptime** → can hint at patch cadence (a broker up for months hasn't been restarted/updated recently)

None of this requires touching device data or publishing anything. It's pure reconnaissance value sitting behind zero credentials.

**This directly mirrors the Day 10 lab** (auth bypass + `$SYS`/wildcard topic enumeration against a deliberately weak broker) — except this is a live, unowned broker rather than a lab construct.

---

## Finding 3 — Properly Secured Broker (Contrast Case)

**Host:** `102.210.149.99` (New IP First Block2, Kenya — Konza)
**MQTT Connection Code: 5** (Not Authorized)

Per the MQTT 3.1.1 spec, CONNACK return codes include:

| Code | Meaning |
|---|---|
| 0 | Connection Accepted |
| 1 | Refused — unacceptable protocol version |
| 2 | Refused — identifier rejected |
| 3 | Refused — server unavailable |
| 4 | Refused — bad username or password |
| 5 | Refused — not authorized |

This broker returned code 5 to Shodan's unauthenticated probe attempt and rejected the connection outright — no `$SYS` data or any other information was exposed. This is the behavior expected of a correctly configured broker requiring authentication.

**This is roughly the response my own Day 8/9 broker would give** to an unauthenticated probe: connection refused, no data leaked, before TLS/mTLS is even considered. It's a useful baseline to show what "doing it right" looks like from the outside.

---

## Side-by-Side Summary

| | 102.68.86.115 | 102.210.149.99 |
|---|---|---|
| CONNACK code | 0 (Accepted) | 5 (Not Authorized) |
| Auth required? | No | Yes |
| `$SYS` data exposed? | Yes — full broker telemetry | No |
| Real-world risk | High — open recon + potential pub/sub access | Low — connection rejected before any data exchange |

---

## Ethics Statement

- All findings in this writeup were obtained by querying Shodan's existing, passively-collected index — no scans or connection attempts were initiated by me against any of the listed hosts.
- I did not connect to, authenticate against, subscribe to, or publish to any live broker discussed here, including the open/misconfigured one.
- No further enumeration (e.g. actively probing for credentials, testing the code-4 "bad password" case, or exploring beyond what Shodan's banner already shows) was attempted.
- Unauthorized access to a computer system — including a misconfigured one — is illegal under Kenya's Computer Misuse and Cybercrimes Act and equivalent laws elsewhere. The value of this exercise is in demonstrating what's discoverable *without* crossing that line.

---

## Tie-Back to Prior Work

| Prior artifact | Connection to this finding |
|---|---|
| Day 10 — MQTT attacks on a weak broker | Finding 2 (open broker, code 0) is the same failure mode observed live, at internet scale, on infrastructure I don't own |
| Day 8 — Server TLS | The exposure in Finding 2 happens over plaintext MQTT; TLS alone wouldn't prevent this specific leak, since `$SYS` exposure is an authorization gap, not an encryption gap — worth noting as a scope boundary of TLS |
| Day 9 — mTLS | Finding 3 (code 5) is roughly the minimum bar my own broker clears before mTLS is even added — auth is the first gate, encryption is the second |

This completes the Phase 1–2 attack-side backfill (MQTT attacks, firmware analysis, Shodan OSINT), giving the banked TLS/mTLS work a proper "before" to sit alongside.
