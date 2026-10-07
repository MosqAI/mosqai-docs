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
    "fan": "standin",
    "uv": "standin",
    "co2": false,
    "temperature": false,
    "power_monitor": false,
    "rgb_led": false,
    "buzzer": false,
    "button": false
  }
}
```
`"standin"` = the command is accepted and drives a test output, but no real
actuator is attached. Apps must label such outputs clearly.
The Last Will payload is `{ "state": "offline" }`.

**heartbeat**
```json
{
  "ts": "2026-10-07T10:00:00Z",
  "uptime_s": 3600,
  "rssi": -61,
  "free_heap": 182000,
  "free_psram": 3900000,
  "camera": "ok",
  "reset_reason": "POWERON",
  "temp_c": null,
  "voltage_v": null,
  "current_a": null,
  "mode": "AUTOMATIC"
}
```

**state**
```json
{ "ts": "...", "mode": "MANUAL", "fan": true, "uv": false, "co2": null, "camera": "ok" }
```

**command**
```json
{ "command_id": "c1b9…uuid", "type": "SET_FAN", "params": { "on": false }, "issued_at": "...", "expires_at": "..." }
```
Initial types: `SET_MODE`, `SET_FAN`, `SET_UV`, `SET_CO2`, `CAPTURE_IMAGE`, `SYNC_SCHEDULE`, `REBOOT`.

**ack**
```json
{ "command_id": "c1b9…", "result": "OK", "actual_state": { "fan": false }, "verification": "pin_readback", "error": null, "ts": "..." }
```
`verification`: `pin_readback` (Phase A: output pin state read back) |
`sensor` (Phase B: confirmed by current draw/tachometer) | `none`.
A command for a capability the device doesn't have returns `REJECTED`
with `error: "UNSUPPORTED"`.
`result`: `OK` | `FAILED` | `REJECTED` (invalid or expired) | `DUPLICATE` (already applied; returns the same state).

## Open questions

1. Heartbeat interval (30 s?) and offline timeout (3 missed beats?).
2. Device credentials: username/password per device vs. X.509 client certificates.
3. ~~Fan health source~~: Phase A uses pin readback; Phase B decides tachometer vs current draw.
4. Image upload auth: same device credential, or short-lived upload token requested over MQTT?
