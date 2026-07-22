# IoT Security Lab

Self-directed IoT security curriculum, run alongside my final-year BSc EEE
(Light Current) coursework. Hands-on work: protocol security, traffic
analysis, vulnerability research, and configs from a home lab.

## Tools

- **Wireshark** — packet capture and traffic analysis
- **Mosquitto** — MQTT broker for hands-on protocol testing
- **Wokwi** — IoT device/circuit simulation
- **MQTTX** — MQTT client for pub/sub and auth testing

## Topics Covered

| Week | Topic | Artifact |
|------|-------|----------|
| 1 | OWASP IoT Top 10 review | — |
| 1 | CVE research: TP-Link TL-WR841N | [`cve-research/tp-link-tl-wr841n.md`](./cve-research/tp-link-tl-wr841n.md) |
| 2 | MQTT traffic capture (Wireshark + Mosquitto) | [`traffic-analysis/mqtt-capture-writeup.md`](./traffic-analysis/mqtt-capture-writeup.md) |
| 2 | Mosquitto username/password auth | [`mqtt-broker/auth-setup-notes.md`](./mqtt-broker/auth-setup-notes.md) |
| 3 | TLS for MQTT (cert gen, port 8883) | [`mqtt-broker/tls/setup-steps.md`](./mqtt-broker/tls/setup-steps.md) |
| 3 | Mutual TLS (client cert auth) | [`mqtt-broker/tls/mutual-tls-notes.md`](./mqtt-broker/tls/mutual-tls-notes.md) |

*(Updated as the curriculum progresses — links point to writeups added along the way.)*

## Structure

```
iot-security-lab/
├── README.md
├── mqtt-broker/
│   ├── mosquitto.conf          # sanitized config, no real credentials
│   ├── auth-setup-notes.md
│   └── tls/
│       ├── ca.crt
│       ├── setup-steps.md
│       └── mutual-tls-notes.md
├── traffic-analysis/
│   └── mqtt-capture-writeup.md
└── cve-research/
    └── tp-link-tl-wr841n.md
```

## Notes

- No real credentials, keys, or private cert material are committed —
  configs are sanitized and any certs included are self-signed lab certs.
- This repo tracks a self-study curriculum, not a production deployment;
  writeups favor "what I did and why" over polish.
