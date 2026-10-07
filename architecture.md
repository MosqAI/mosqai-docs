# Architecture

## System shape

```
 ESP32-CAM device ──MQTT over TLS (telemetry, events, acks)──┐
        ▲                                                    │
        │ commands (MQTT)          image upload (HTTPS)      ▼
        └────────────────────────────────────────── MOSQAI BACKEND (NestJS)
                                                     │  ├── PostgreSQL (Prisma)  ← source of truth
                                                     │  ├── Object storage       ← original images
                                                     │  ├── AI service (Python)  ← detections
                                                     │  ├── Analytics / alerts
                                                     │  └── REST + WebSockets
                                                     ▼
                                     Mobile app (owners)   Authority web (RBAC, areas)
```

**Rules**

1. The backend is the single source of truth. Apps never talk to a device directly.
2. Devices talk only to the MQTT broker and the backend's upload endpoint.
3. The AI service is called by the backend. It never talks to devices or apps.
4. Every piece of state shown in an app is backed by a database row the backend wrote.

## Sources of truth

| Fact | Owner | Notes |
| --- | --- | --- |
| Physical hardware state (fan on, UV on, mode) | **Device** reports it, backend records the *confirmed* value | The backend never assumes a command succeeded |
| Desired state / pending commands | Backend (`device_commands`) | |
| Users, roles, ownership, areas | Backend | |
| Original images | Object storage, referenced by `images` row | Never modified or deleted by the pipeline |
| Detections | Backend (`ai_results`, `ai_detections`), produced by the AI service | Always linked to `image_id` and `model_version` |

## Command lifecycle

A command is only successful when the device confirms it.

```
App ──POST /devices/:id/commands──▶ Backend: row status=PENDING
Backend ──publish cmd──▶ Device                    status=SENT
Device: applies change, re-reads hardware state
Device ──publish ack {ok|failed, actual_state}──▶ Backend
Backend: status=CONFIRMED|FAILED, updates confirmed device state
Backend ──WebSocket──▶ App shows the confirmed state
No ack before the timeout → status=TIMED_OUT and the app shows "unconfirmed"
```

Commands carry a unique `command_id` so retries are idempotent on the device.

## Control modes

- `MANUAL`: commands directly set actuators.
- `AUTOMATIC`: the device runs the schedule/rules the backend synced to it, so it
  keeps working offline.
- A mode switch is itself a command and follows the lifecycle above. When the
  device returns to `AUTOMATIC` it re-evaluates the current rule for "now". It
  does not resume a stale state.

## Image → AI pipeline

```
Device captures → HTTPS upload (device credential) → storage (original kept)
→ images row → job queued → AI service → ai_results + ai_detections
→ analytics aggregates → WebSocket / REST to apps
```

## Data honesty: "no detections" ≠ "no mosquitoes"

Analytics must label every time bucket for every device with a coverage state:

| State | Meaning |
| --- | --- |
| `ACTIVE` | Device online and reporting; counts are meaningful (including 0) |
| `NO_DATA` | Device offline, faulted, or not reporting; counts are **unknown** |

Area aggregates show coverage ("8 of 10 devices reporting") next to counts.

## Device status (local LED)

| Colour | Meaning |
| --- | --- |
| Green | Normal |
| Blue | Connecting |
| Amber | Warning |
| Red | Fault |
| Off | Device off |

Hardware note: high-current loads (fan, UV-A LEDs, CO₂ valve/pump) go through
MOSFET or driver stages. They are never driven directly from ESP32 GPIO pins.

## Hardware phases

The software must work with whatever hardware a device actually has. Each device
reports its **capabilities**, and anything it doesn't have is shown as *not
available*, never as a fake or zero value.

### Phase A — ESP32-CAM only (current)

The team currently has only an AI-Thinker ESP32-CAM: no fan, UV-A, CO₂,
temperature, voltage/current sensors, RGB LED or buzzer.

| Function | Phase A implementation |
| --- | --- |
| Connectivity, identity, MQTT, heartbeat, online/offline | Full |
| Commands + ack | `SET_FAN` / `SET_UV` drive stand-in outputs: the onboard flash LED (GPIO 4) and GPIO 13/14 (free when the SD card is unused). The device reads the pin back before acking. |
| Camera capture + upload | Full (OV2640, PSRAM) |
| Health | RSSI, uptime, free heap/PSRAM, camera init status, reset reason. Temperature, voltage and current are reported as `null`. |
| Status indication | Onboard red LED (GPIO 33) blink codes instead of RGB; no buzzer |
| Manual / automatic mode, schedules | Full (software only) |

Constraints:

- **No analog sensing while Wi-Fi is on.** The free pins are on ADC2, which
  the Wi-Fi radio blocks, so voltage/current sensing needs an external I²C ADC
  or monitor (e.g. INA219) in Phase B.
- **Stand-in outputs only prove pin state.** In Phase A the ack's
  `verification` is `pin_readback`. Real verification (current draw,
  tachometer) arrives with the Phase B hardware.
- **Power:** use a stable 5 V supply of at least 2 A; Wi-Fi and the camera together cause brownout resets on weak supplies.
- **Programming:** an ESP32-CAM-MB or FTDI adapter is needed (no USB on the board).
- **AI:** the OV2640 is fixed-focus, 2 MP. Counting mosquitoes on a lit catch
  surface is realistic. Genus classification may not be. Train on images from
  this camera.

### Phase B — full device

Adds MOSFET/driver stages for fan, UV-A and CO₂; a temperature sensor; an INA219
(or similar) for voltage/current; RGB LED; buzzer; button. Because firmware and
backend already treat these as optional capabilities, Phase B is additive: no
contract changes are needed beyond new capability flags.

## Security baseline

- Users: JWT access tokens plus rotating refresh tokens; RBAC roles (`OWNER`, `AUTHORITY_VIEWER`, `AUTHORITY_ADMIN`, `ADMIN`).
- Devices: per-device credentials, MQTT over TLS, broker ACLs restricting each device to its own topics.
- Images are served only through authorised, short-lived URLs.
- Authority users see device IDs and areas by default, not owner identities.
- Commands and permission changes are written to an audit log.
- Secrets live only in environment variables. Only `.env.example` files are committed.
