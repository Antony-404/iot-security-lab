# Wireshark Comparison: MQTT without Client Certificate vs. mTLS

**Day 9 — Mutual TLS (mTLS) for MQTT**

## Objective

Building on the TLS setup from Day 8 (server-side TLS on port 8883), this
capture demonstrates the effect of enabling `require_certificate true` on
the Mosquitto broker: connections without a valid client certificate are
rejected during the TLS handshake, before any MQTT-level authentication
or messaging can occur.

## Setup

- Mosquitto 2.1.2, listener on 8883 with:
  - `require_certificate true`
  - `use_identity_as_username true`
  - `cafile` pointing to the lab's CA (`ca.crt`)
- Client certificate (`client.crt`) generated and signed by the same CA,
  CN = `mqtt-client-01`
- Capture taken on the Npcap "Adapter for loopback traffic capture" (same
  as Day 8), filtered to `tcp.port == 8883`
- Two connection attempts made via MQTTX:
  1. TLS enabled, no client certificate configured
  2. TLS enabled, client certificate + key configured

## A note on TLS 1.3 and what's actually visible

Both captures negotiate **TLS 1.3**. Unlike TLS 1.2, TLS 1.3 encrypts
essentially everything after `Server Hello` — the server's certificate,
the client's certificate, and all handshake-finish messages are sent as
opaque `Application Data`, not labeled by message type. This means
Wireshark does not show a distinctly labeled "client sends Certificate"
frame the way it might for a TLS 1.2 capture. The distinguishing signal
instead is the **shape of the exchange**: whether the client sends a
follow-up `Application Data` bundle back to the server after receiving
the server's handshake bundle, or whether the client aborts the
connection instead.

## Results

### Attempt 1 — No client certificate (rejected)

| Frame | Direction | Content |
|---|---|---|
| 109-113 | Client -> Server | TCP handshake, `Client Hello` |
| 114 | Server -> Client | `Server Hello, Change Cipher Spec, Application Data x4` (server's cert bundle) |
| 116 | Client -> Server | `FIN, ACK` — client terminates |
| 119 | Client -> Server | `RST, ACK` — hard reset |

The server completed its side of the handshake, but the client had no
certificate to present back. Without it, the client could not complete
the TLS 1.3 handshake and dropped the connection itself (FIN, then RST).
No MQTT traffic was ever exchanged. This matches the Mosquitto broker log
for this attempt:

```
OpenSSL Error while trying to get the error[0]: error:0A000126:SSL routines::unexpected eof while reading
Client ::1 [::1:53523] disconnected: protocol error.
```

### Attempt 2 — With client certificate (accepted)

| Frame | Direction | Content |
|---|---|---|
| 59-61 | Client -> Server | TCP handshake |
| 62 | Client -> Server | `Client Hello` |
| 64 | Server -> Client | `Server Hello, Change Cipher Spec, Application Data x4` (server's cert bundle) |
| 66-67 | Client -> Server | `Change Cipher Spec, Application Data` — client's certificate, CertificateVerify, and Finished |
| 68-73 | Both directions | `Application Data` — MQTT session traffic (CONNECT/CONNACK and beyond) inside the established tunnel |

Here the client responds to the server's bundle with its own
`Application Data` round trip — this is the client presenting its
certificate and completing mutual authentication. The connection then
proceeds normally, carrying actual MQTT traffic, with no FIN/RST. The
Mosquitto log confirms the client authenticated using its certificate
identity rather than a username/password:

```
New client connected from ::1:52295 as mqtt-client-01 (p5, c1, k60, u'mqtt-client-01').
Client mqttx_eea60d72 negotiated TLSv1.3 cipher TLS_AES_256_GCM_SHA384
```

## Why this matters

Server-only TLS (Day 8) proves the *server's* identity to the client and
encrypts the channel, but the broker still has no cryptographic proof of
who is connecting — only whatever the MQTT-level auth (e.g. password)
provides, and that check happens only after the tunnel is already up.

With `require_certificate true`, the broker demands proof of client
identity as part of the TLS handshake itself. An unauthorized client is
rejected before the MQTT layer is ever reached — visible here as an
immediate FIN/RST with zero MQTT bytes exchanged, versus a completed
handshake and live session for a client holding a CA-signed certificate.
This moves client authentication from the application layer down into
the transport layer, closing the gap where a connection could be
encrypted but still not truly identity-verified.
