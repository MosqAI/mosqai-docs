# MQTT contract — v0 DRAFT

> Status: **DRAFT**. RavynX0 and Hope664 must both review and approve before M1 code is
> written. Change via PR to `mosqai-docs` only.

## Conventions

- Broker: Mosquitto (dev), TLS on port 8883. Plain 1883 only on a local dev machine.
- Each device authenticates with its own credentials; broker ACL limits it to `mosqai/v1/devices/<its id>/#`.
- Payloads are JSON (UTF-8). Timestamps are ISO-8601 UTC. Images are **not** sent over MQTT.
- `device_id` format: `MSQ-` + zero-padded number, e.g. `MSQ-000001`.

## Topics

| Topic | Direction | QoS | Retained | Purpose |
| --- | --- | --- | --- | --- |
| `mosqai/v1/devices/{id}/status` | device → backend | 1 | yes | `online` / `offline` (offline set as Last Will) |
| `mosqai/v1/devices/{id}/heartbeat` | device → backend | 0 | no | Periodic health snapshot |
| `mosqai/v1/devices/{id}/state` | device → backend | 1 | yes | Actual actuator state whenever it changes |
| `mosqai/v1/devices/{id}/events` | device → backend | 1 | no | Faults, button presses, boot |
| `mosqai/v1/devices/{id}/commands` | backend → device | 1 | no | Commands |
| `mosqai/v1/devices/{id}/commands/ack` | device → backend | 1 | no | Command results |

## Payloads (proposed)

Fields for hardware a device doesn't have are `null`, never `0` or a guess.
See *Hardware phases* in `architecture.md`.

**status** (retained; `online` payload also carries capabilities)
```json
{
  "state": "online",
  "fw": "0.1.0",
  "hw": "esp32-cam-ai-thinker",
  "capabilities": {
    "camera": true,
    "camera_led": true,
    "fan": true,
    "fan_tach": false,
    "uv": true,
    "buzzer": false,
    "co2_chamber": "passive",
    "temperature": false,
    "power_monitor": false,
    "co2_sensor": false,
    "rgb_led": false,
    "button": false
  }
}
```
`co2_chamber: "passive"` = yeast + sugar chamber with no control or sensor; the
backend estimates its state from refill time (see `architecture.md`).
The Last Will payload is `{ "state": "offline" }`.

**heartbeat**
```json
{
  "ts": "2026-10-07T10:00:00Z",
  "uptime_s": 3600,
  "rssi": -61,
  "free_heap": 182000,
  "free_psram": 3900000,
  "reset_reason": "POWERON",
  "reboots_24h": 0,
  "mode": "AUTOMATIC",
  "camera": "ok",
  "fan_rpm": null,
  "last_self_test": { "ts": "...", "uv": "ok", "camera_led": "ok" },
  "temp_c": null,
  "voltage_v": null,
  "current_a": null
}
```
`fan_rpm` is `null` while the fan has no tachometer wire (current 2-wire fan). `buzzer` becomes `true` once one is fitted.

**state** (retained, published on every change)
```json
{ "ts": "...", "mode": "MANUAL", "fan": true, "uv": false, "camera_led": false, "camera": "ok" }
```

**events**
```json
{ "ts": "...", "type": "FAN_STALLED", "severity": "FAULT", "detail": { "rpm": 0 } }
```
Event types: `BOOT`, `FAN_STALLED`, `UV_FAULT`, `CAMERA_LED_FAULT`,
`CAMERA_FAULT`, `POWER_UNSTABLE` (brownout reset), `SELF_TEST_DONE`.
Severity: `INFO` | `WARNING` | `FAULT`.

**command**
```json
{ "command_id": "c1b9…uuid", "type": "SET_FAN", "params": { "on": false }, "issued_at": "...", "expires_at": "..." }
```
Types: `SET_MODE`, `SET_FAN`, `SET_UV`, `SET_CAMERA_LED`, `BUZZ`
(`params.pattern`: `ACK`, `WARNING`, `FAULT`, `LOCATE`), `CAPTURE_IMAGE`,
`SELF_TEST`, `SYNC_SCHEDULE`, `REBOOT`.
There is no CO₂ command: the chamber is passive.

**ack**
```json
{ "command_id": "c1b9…", "result": "OK", "actual_state": { "fan": true, "fan_rpm": null }, "verification": "pin_readback", "error": null, "ts": "..." }
```
`verification`: `tach` (fan RPM measured) | `camera_check` (brightness change
seen by the camera) | `pin_readback` (output pin read back only) | `none`.
A command for a capability the device doesn't have returns `REJECTED`
with `error: "UNSUPPORTED"`. If verification fails, the result is `FAILED`
and `actual_state` reports what was observed.
`result`: `OK` | `FAILED` | `REJECTED` (invalid or expired) | `DUPLICATE` (already applied; returns the same state).

## Open questions

1. Heartbeat interval (30 s?) and offline timeout (3 missed beats?).
2. Device credentials: username/password per device vs. X.509 client certificates.
3. ~~Fan type~~: the current fan is 2-wire, so verification is `pin_readback` until a 3-wire fan is fitted.
5. Self-test brightness thresholds for UV-A and the camera LED: calibrate on the real enclosure.
6. CO₂ chamber status thresholds (days active/declining): tune from real refills.
4. Image upload auth: same device credential, or short-lived upload token requested over MQTT?
