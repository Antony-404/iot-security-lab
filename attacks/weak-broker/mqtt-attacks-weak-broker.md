# MQTT Attack Techniques — Weak Broker (Day 10)

## Overview

This session demonstrates three common MQTT attack techniques against a
deliberately unsecured Mosquitto broker instance, kept separate from the
TLS/mTLS-secured broker built in Day 8/9. This serves as the "before" half
of a before/after security narrative — the attacks demonstrated here are
exactly what the TLS/mTLS controls in `wireshark-tls-comparison.md` and
`wireshark-mtls-comparison.md` are designed to prevent.

**Attacks demonstrated:**
1. Authentication bypass
2. Topic enumeration (`#` wildcard)
3. Message injection (retained messages)

## Setup

A second, isolated Mosquitto instance was configured specifically to be
vulnerable, separate from the production/secured config:

**`weak-mosquitto.conf`**
```
listener 1883
allow_anonymous true
```

- `listener 1883` — plaintext MQTT default port, isolated from the secured
  broker's 8883 listener
- `allow_anonymous true` — the core vulnerability: any client can connect
  with no username, password, or certificate. This is the most common
  real-world MQTT misconfiguration and is exactly what Shodan `port:1883`
  scans find at scale.

Run with:
```bash
"/c/Program Files/mosquitto/mosquitto.exe" -c weak-mosquitto.conf -v
```

## Attack 1: Authentication Bypass

A client subscribed and another published, both with zero credentials —
no `-u`/`-P` flags, no certs:

```bash
# Subscriber
mosquitto_sub -h localhost -p 1883 -t "test/topic" -v

# Publisher
mosquitto_pub -h localhost -p 1883 -t "test/topic" -m "hello from attacker"
```

**Result:** Broker log confirmed the connection, subscribe, and publish
were all accepted and relayed with no identity check at any point:

```
Received CONNACK to auto-AAA3DA33-...
Received PUBLISH from auto-AAA3DA33-... (d0, q0, r0, m0, 'test/topic', ... (19 bytes))
Sending PUBLISH to auto-65434CD2-... (d0, q0, r0, m0, 'test/topic', ... (19 bytes))
```

The message `hello from attacker` was received by the subscriber
end-to-end, with the broker never verifying who either party was.

## Attack 2: Topic Enumeration

Rather than knowing a topic name in advance, an attacker can subscribe to
the multi-level wildcard `#` to discover every topic being published on
the broker — mapping out a target's full MQTT namespace passively.

```bash
mosquitto_sub -h localhost -p 1883 -t "#" -v
```

Three simulated device topics were then published by a separate client:

```bash
mosquitto_pub -h localhost -p 1883 -t "home/livingroom/temp" -m "22.5"
mosquitto_pub -h localhost -p 1883 -t "device/status" -m "online"
mosquitto_pub -h localhost -p 1883 -t "admin/creds" -m "admin:password123"
```

**Result:** All three topics — including `admin/creds` — appeared on the
single anonymous `#` subscription, despite never being subscribed to by
name. This illustrates two risks at once: broker-wide topic discovery
with a single connection, and the real-world anti-pattern of publishing
sensitive data (credentials) in plaintext over MQTT topics.

## Attack 3: Message Injection (Retained Messages)

An attacker can impersonate a legitimate device by publishing to a known
topic. Using the `-r` (retain) flag, the forged message persists on the
broker and is delivered automatically to any future subscriber — even
one that connects long after the injection:

```bash
mosquitto_pub -h localhost -p 1883 -t "device/status" -m "online" -r
```

A fresh subscriber connecting afterward, with no prior knowledge of this
publish, immediately received the forged state:

```bash
mosquitto_sub -h localhost -p 1883 -t "device/status" -v
```

**Result:** The injected message was delivered as authoritative current
state to a brand-new client. In a real deployment this could forge
device status, sensor readings, or actuator commands (e.g. `unlock`,
`shutdown`) — the payload is entirely attacker-controlled.

## Wireshark Evidence

Packet capture on `port 1883` (loopback, Npcap adapter) during the
`admin/creds` publish confirms all MQTT traffic is fully readable in
plaintext, with no decryption required:

- **Packet list:** `MQTT 96 Publish Message [admin/creds]` — Wireshark
  decodes the topic name directly off the wire
- **Hex dump:** raw ASCII bytes spell out `admin/creds` and
  `admin:password123` in the packet payload
- **MQTT layer (expanded):** structured fields show
  `Topic: admin/creds`, `Message: admin:password123` (decoded as ASCII)

*(Screenshots: packet list + hex dump, expanded MQTT PUBLISH layer,
ASCII-decoded message view — see `iot-screenshots/`)*

## Comparison to Secured Broker (Day 8/9)

| | Weak Broker (1883) | Secured Broker (8883, mTLS) |
|---|---|---|
| Authentication | None (`allow_anonymous true`) | Client cert required (`require_certificate true`) |
| Traffic visibility | Fully plaintext — topic + payload readable in Wireshark | Opaque `Application Data` after TLS handshake |
| Topic discovery | Trivial via `#` wildcard | Still possible post-auth, but requires a valid cert first |
| Message injection | Trivial — any client can publish to any topic | Requires a valid, signed client cert |

## Key Takeaways

- `allow_anonymous true` is a single-line misconfiguration with
  broker-wide consequences — no auth means no accountability for any
  connect, subscribe, or publish action.
- The `#` wildcard turns "guess the topic name" into "see everything,"
  making broker-wide reconnaissance trivial for an unauthenticated
  client.
- Retained messages (`-r`) let an attacker plant persistent forged state
  that outlives their own connection — future clients trust it as fact.
- None of these attacks required any protocol-level exploit — they're
  all standard MQTT features used as intended, against a broker with no
  access control. This is why TLS/mTLS (Day 8/9) matters: it doesn't
  patch a bug, it closes the identity gap these attacks all rely on.
