# ACL Enforcement — Scoped Authorization on Top of mTLS

## Objective
mTLS (Day 9) proves *identity* — a client must present a valid signed cert
to connect at all. It does not restrict what an authenticated client can
do once connected. This lab adds Mosquitto ACLs to enforce least-privilege
topic access per identity, using the cert CN (via `use_identity_as_username`)
as the authorization key.

## Setup
- Two client identities, each with its own CA-signed cert:
  - `sensor-livingroom` — intended to publish sensor data only, no read access
  - `admin-dashboard` — intended to have full read/write across the topic tree
- `acl_file` referencing `acl.conf`, default-deny (no rule = no access):

user sensor-livingroom
topic write home/livingroom/#

user admin-dashboard
topic readwrite home/#

- Existing Day 9 mTLS config unchanged: `require_certificate true`,
  `use_identity_as_username true` on the 8883 listener — ACLs match
  against the cert CN with no additional auth mechanism needed.

## Test Matrix & Results

| Client | Action | Expected | Result |
|---|---|---|---|
| sensor-livingroom | publish `home/livingroom/test` | Allowed | Allowed |
| sensor-livingroom | publish `home/frontdoor/lock` | Denied | Denied |
| sensor-livingroom | subscribe `home/livingroom/` | Denied (write-only) | Denied |
| admin-dashboard | subscribe `home/#` | Allowed | Allowed |
| admin-dashboard | publish `home/frontdoor/lock` | Allowed | Allowed |

## Evidence

**sensor-livingroom — denied subscribe (write-only enforcement):**

1789232651: Received UNSUBSCRIBE from mqttx_771631d3
1789232651:     home/livingroom/
1789232651: mqttx_771631d3 home/livingroom/
1789232651: Sending UNSUBACK to mqttx_771631d3

**sensor-livingroom — allowed publish (in-scope topic):**

1789232687: Received PUBLISH from mqttx_771631d3 (d0, q0, r0, m0, 'home/livingroom/test', ... (19 bytes))

No denial logged — publish accepted per `write` grant on `home/livingroom/#`.

**sensor-livingroom — denied publish (out-of-scope topic):**

1789232834: Denied PUBLISH from mqttx_771631d3 (d0, q0, r0, m0, 'home/frontdoor/lock', ... (19 bytes))

**admin-dashboard — allowed subscribe (full tree read):**

1789233010: Received SUBSCRIBE from mqttx_9e80a4d3
1789233010:     home/# (QoS 0)
1789233010: mqttx_9e80a4d3 0 home/#
1789233010: Sending SUBACK to mqttx_9e80a4d3

**admin-dashboard — allowed publish, echoed back via own subscription:**

1789233025: Received PUBLISH from mqttx_9e80a4d3 (d0, q0, r0, m0, 'home/frontdoor/lock', ... (19 bytes))
1789233025: Sending PUBLISH to mqttx_9e80a4d3 (d0, q0, r0, m0, 'home/frontdoor/lock', ... (19 bytes))

## Why this matters
Day 10's attack lab showed that an unauthenticated broker lets any client
enumerate topics and inject messages freely. mTLS alone closes the
authentication gap but does not scope what a *legitimate, authenticated*
device is permitted to do — a compromised or misconfigured sensor would
still be able to read or write any topic on the broker. ACLs close that
second gap: even a fully authenticated `sensor-livingroom` identity is
cryptographically confirmed *and* contained to its own topic namespace,
completing the least-privilege model alongside mTLS.

## Key learnings
- `use_identity_as_username` is what makes ACLs work off cert CNs directly
  — no separate username/password layer needed on top of mTLS
- Default-deny requires no explicit "deny all" line; an identity with no
  matching rule is blocked automatically
- MQTTX's subscription UI can appear to show an "active" subscription
  even when the broker denied it — the broker log, not the client UI,
  is the source of truth for ACL enforcement`
