# Roadmap

Milestones are sequential. Each later one depends on the one before it working end to end.

## M0 — Foundation (current)

- [ ] GitHub org, repos, `main`/`develop` branches, branch protection
- [ ] Project board + labels + issues
- [ ] Agree MQTT contract v0 (`contracts/mqtt.md`)
- [ ] Agree initial data model (ADR)
- [ ] Inspect Figma design → screen/component inventory (`design/`)
- [ ] Local dev stack: Postgres + Mosquitto via docker-compose

## M1 — Core communication loop (the first real milestone)

ESP32 → MQTT → backend → PostgreSQL, and backend → MQTT → ESP32.

**Backend:** init NestJS + Prisma; device + credential model; device registration;
MQTT client; heartbeat ingestion; online/offline detection (heartbeat timeout);
command create/send; ack handling + timeout; tests for each.
**Device:** init firmware (PlatformIO); Wi-Fi + reconnect; device identity;
MQTT over TLS; heartbeat; command handler for fan/UV with state verification;
ack publishing; RGB status LED.
**Infra:** compose with Postgres + Mosquitto (TLS + ACL); device simulator.

## M2 — Image → AI → database

**Device:** camera capture, HTTPS upload, retry queue.
**Backend:** upload endpoint, storage adapter, `images` table, AI job dispatch,
result storage.
**AI:** service skeleton (FastAPI), preprocessing, YOLO detection, confidence,
bounding boxes, model versioning, evaluation script.

## M3 — Mobile app (owner)

Auth, device pairing, dashboard, confirmed state, manual control, mode switch,
health, latest images + detections.

## M4 — Authority web

Auth + RBAC, areas, device map, device health, area activity with coverage,
AI evidence.

## M5 — Analytics, alerts, automation

Daily/weekly/monthly trends, peak activity, alert rules (offline, camera, fan,
power, temperature), notifications, schedules/automation rules synced to device.

## M6 — Hardening & deployment

Staging + production environments, CI/CD deploys, backups, monitoring, rate
limiting review, audit log review, load test with simulated device fleet.
