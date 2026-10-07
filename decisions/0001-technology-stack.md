# ADR 0001 — Technology stack

- Status: Accepted
- Date: 2026-10-07

## Context

Earlier documents (`MosqAI_SRS.docx`, project description) list Express,
MongoDB/Firebase, TensorFlow, React + Tailwind, and a kill/release mechanism.
The current engineering brief defines a different stack and device scope.

## Decision

| Area | Choice |
| --- | --- |
| Backend | Node.js, NestJS, TypeScript, Prisma, PostgreSQL, MQTT, REST, WebSockets |
| AI | Python, PyTorch, YOLO, OpenCV (served as a separate service) |
| Mobile | React Native, Expo, TypeScript |
| Authority web | Next.js, React, TypeScript, ECharts or Recharts, MapLibre |
| Device | ESP32-CAM, C/C++ (PlatformIO, Arduino framework), MQTT over TLS |
| Infra | Docker, Docker Compose, GitHub Actions |

PostgreSQL is the single database. MongoDB is not used, so there is one source of
truth. Time-series volume can be handled later with partitioning or TimescaleDB
if needed.

## Open scope question

The SRS mentions "harmful mosquitoes neutralized; non-harmful insects released",
plus heat and chemical lures. The current brief describes UV-A + CO₂ + fan
capture only. Confirm whether kill/release is in scope before designing device
commands for it.
