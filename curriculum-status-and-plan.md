# IoT Security Curriculum — Status & Plan

_Last updated: Day 15 (ACL enforcement complete)_

## Where We Actually Are

Progress has followed a protocol-security-first path, diverging from
the original 28-week roadmap's sequencing:

| Day | Topic | Status |
|---|---|---|
| Week 1 | OWASP IoT Top 10, CVE research on TP-Link TL-WR841N | Done |
| Week 2 | MQTT traffic capture (Wireshark), Mosquitto username/password auth | Done |
| Day 8 | TLS for MQTT — CA + server cert, listener on 8883, plaintext vs TLS Wireshark comparison | Done, pushed |
| Day 9 | Mutual TLS (mTLS) — client cert generation/signing, require_certificate, handshake comparison | Done, pushed |
| Day 10 | MQTT attacks on weak broker — auth bypass, topic enumeration, message injection | Done, pushed |
| Day 12 | Firmware analysis — OpenWrt squashfs extraction, empty root password hash finding | Done, pushed |
| Day 13 | Shodan OSINT — port:1883 country:"KE" recon, misconfigured broker found | Done, pushed |
| Day 14 | Linux Mint migration — certs re-signed, git re-cloned, mosquitto.conf paths fixed | Done, pushed |
| Day 15 | Mosquitto ACLs — scoped topic authorization on top of mTLS identity, default-deny acl_file, full allow/deny test matrix verified | Done, pushed |

**Repo:** iot-security-lab — github.com/Antony-404/iot-security-lab

**Portfolio artifacts banked so far:**
- wireshark-tls-comparison.md (Day 8)
- wireshark-mtls-comparison.md (Day 9)
- attacks/ writeup (Day 10)
- firmware-analysis.md (Day 12)
- shodan-mqtt-osint.md (Day 13)
- acl-enforcement.md (Day 15)

## Status Summary

Phase 1-2 attack-side backfill is complete (Days 10, 12, 13). The
before/after security narrative is intact: unsecured broker attacks
(Day 10) -> firmware/OSINT recon (Days 12-13) -> TLS/mTLS defense
(Days 8-9, built early but now contextualized) -> ACL scoped
authorization (Day 15) as the natural extension of mTLS identity.

## On the Horizon

Pending decision at start of next session:
1. Phase 3 hardware check — NodeMCU/ESP8266 availability
2. Phase 4 FYP/eJPT alignment — beginning final phase work

## Full Original Roadmap (for reference)


- **Phase 0 (Week 1):** Environment setup — Mosquitto, MQTTX, Node-RED,
  Wokwi familiarization, GitHub/Shodan accounts, download router firmware
- **Phase 1 (Weeks 2-6):** IoT fundamentals + attack surface — MQTT
  protocol deep dive, Wireshark plaintext capture, CoAP/HTTP context,
  Shodan port scanning, Binwalk firmware extraction, Node-RED pipelines
- **Phase 2 (Weeks 7-14):** Hands-on attacks — MQTT auth bypass/topic
  enum/injection, deeper firmware analysis + CVE research, network-layer
  attacks, web interface attacks on IoT admin panels
- **Phase 3 (Weeks 15-20):** Hardware arrives (NodeMCU/ESP8266) —
  physical lab replication, attacking own device, UART exploration,  MQTT-over-TLS as the defensive capstone
- **Phase 4 (Weeks 21-28):** FYP alignment, eJPT certification prep

**End-state portfolio target:** 6+ GitHub writeups, eJPT certification,
TryHackMe SOC path completed, hands-on ESP8266 hardware experience.
