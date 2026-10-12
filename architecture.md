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

## Device status (local LED + buzzer)

| State | Phase A: white status LED (GPIO 13) | Buzzer (when fitted) | Phase B: RGB LED |
| --- | --- | --- | --- |
| Normal | Short blink every 5 s | silent | Green |
| Connecting | Fast blink (5 Hz) | silent | Blue |
| Warning | Double blink every 2 s | 2 short beeps once | Amber |
| Fault | Solid on | 3 long beeps, repeated every 5 min until cleared | Red |
| Off | Off | silent | Off |

A command ack gets 1 short beep. `LOCATE` beeps continuously for 10 s, to find a device.

Hardware note: high-current loads (fan, UV-A LEDs) go through
MOSFET or driver stages. They are never driven directly from ESP32 GPIO pins.

## Hardware phases

The software must work with whatever hardware a device actually has. Each device
reports its **capabilities**, and anything it doesn't have is shown as *not
available*, never as a fake or zero value.

### Phase A — current hardware, no sensors

#### Parts in hand (2026-10-07)

ESP32-CAM, power adapter, buck converter (adapter → 5 V for the ESP32-CAM),
capacitor (across the ESP32 5 V/GND to absorb power dips), small white LED,
**2-wire fan** (no tachometer), UV-A LED strip, wires, plus the yeast + sugar CO₂
chamber. There are **no sensors**: no temperature, voltage/current or CO₂ sensor.
Everything below is monitored using only the ESP32-CAM itself.

#### Still to buy before the firmware work

| Item | Why |
| --- | --- |
| ESP32-CAM-MB programmer (or FTDI adapter) | The board has no USB port, so firmware can't be flashed without it |
| 2 × **logic-level** MOSFET modules (AO3400 / IRLZ44N / D4184), **not** IRF520 | Switch the fan and UV-A strip from 3.3 V GPIO |
| 1 × diode (1N5819 / 1N4007) | Flyback protection across the fan motor |
| 220 Ω resistor | For the white status LED |
| Optional: active buzzer + 2N2222 transistor + 1 kΩ resistor | Audible alerts. Firmware reports `buzzer: false` until it's fitted. |
| Optional: 3-wire fan | Lets the device measure RPM and detect a stalled fan |

Power budget: the adapter rating and the fan and UV-strip voltages still need to be confirmed.

ESP32 pins must never power the fan, UV strip or buzzer directly.

#### Pin map (AI-Thinker ESP32-CAM, SD card unused)

| GPIO | Use | Notes |
| --- | --- | --- |
| 14 | Fan on/off (MOSFET gate) | |
| 15 | UV-A on/off (MOSFET gate) | Strapping pin; the pull-down keeps it LOW at boot |
| 13 | White status LED (via 220 Ω) | Visible outside the enclosure. If a 3-wire fan is fitted later, its tachometer moves here and the status LED moves to GPIO 33. |
| 2 | Buzzer (transistor base), when fitted | Strapping pin; must be LOW/floating at boot, so it uses a pull-down |
| 4 | Camera LED (onboard flash) | Already has its own transistor on the board |
| 33 | Onboard red LED (active LOW) | Debug only (hidden inside the enclosure) |
| 12 | Spare, avoid | Strapping pin (flash voltage). HIGH at boot breaks booting. |
| 1 / 3 | Serial debug (UART0) | |

#### How each part is monitored without sensors

| Part | Control | Verification (`verification` value in acks) | Health signal |
| --- | --- | --- | --- |
| Fan (2-wire, current) | `SET_FAN` | `pin_readback`. Apps show "on, not verified". | none (a stalled fan can't be detected) |
| Fan (3-wire, upgrade) | `SET_FAN` | `tach` (RPM measured) | RPM = 0 while ON → `FAN_STALLED` fault |
| UV-A LEDs | `SET_UV` | **Camera self-test** (`camera_check`): frame brightness with UV off vs on must rise above a calibrated threshold | Self-test failed → `UV_FAULT` warning |
| Camera LED | `SET_CAMERA_LED`, automatic during capture | `camera_check` (same brightness-delta method) | Self-test failed → `CAMERA_LED_FAULT` warning |
| Camera | `CAPTURE_IMAGE` | Capture succeeds and the frame is not black, saturated or frozen | Init fail / bad frames → `CAMERA_FAULT` fault |
| Buzzer | `BUZZ` (pattern) | `pin_readback` only (nothing can hear it) | none |
| CO₂ chamber | none (passive fermentation) | not applicable | **Estimated** from time since refill (see below) |
| Power | n/a | n/a | Brownout reset reason + reboot count → `POWER_UNSTABLE` warning |
| Temperature, voltage, current | n/a | n/a | `null`, shown as *not available* |

`SELF_TEST` runs the camera checks on demand. The device also runs them at boot
and once a day (UV and LED pulsed briefly).

#### CO₂ chamber (yeast + sugar)

The chamber is passive: fermentation produces CO₂ continuously and the device
cannot measure or control it. The backend models it instead:

- The owner records a **refill** in the app (`POST /devices/:id/co2-chamber/refills`).
- The backend derives `co2_status` from the time since the last refill:
  `ACTIVE` → `DECLINING` → `EXHAUSTED`, or `UNKNOWN` if no refill has been recorded.
  The thresholds are configurable (starting guess: 0–5 days active, 5–10 declining, then exhausted) and are tuned with real observations.
- Apps always label it **"estimated"**, and an alert reminds the owner to refill.
- Analytics stores the chamber state with each detection, so capture rates can
  later be compared against chamber age.

#### Other constraints

- **No analog sensing while Wi-Fi is on.** The free pins are on ADC2, which the
  Wi-Fi radio blocks.
- **Power:** a stable supply sized for the ESP32-CAM (≥ 2 A at 5 V) **plus** the
  fan and UV-A load, with a common ground. Wi-Fi and the camera together cause
  brownout resets on weak supplies.
- **Programming:** an ESP32-CAM-MB or FTDI adapter is needed (no USB on the board).
- **AI:** the OV2640 is fixed-focus, 2 MP. Counting mosquitoes on a lit catch
  surface is realistic. Genus classification may not be. Train on images from
  this camera.

### Phase B — sensors added later

Possible additions: temperature sensor, INA219 voltage/current monitor, CO₂
sensor, RGB LED, button. Firmware and backend treat every part as an optional
capability, so Phase B only flips capability flags and fills fields that are
`null` today. No contract redesign is needed.

## Security baseline

- Users: JWT access tokens plus rotating refresh tokens; RBAC roles (`OWNER`, `AUTHORITY_VIEWER`, `AUTHORITY_ADMIN`, `ADMIN`).
- Devices: per-device credentials, MQTT over TLS, broker ACLs restricting each device to its own topics.
- Images are served only through authorised, short-lived URLs.
- Authority users see device IDs and areas by default, not owner identities.
- Commands and permission changes are written to an audit log.
- Secrets live only in environment variables. Only `.env.example` files are committed.
