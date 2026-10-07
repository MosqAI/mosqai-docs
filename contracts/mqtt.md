# MQTT contract — v0 DRAFT

> Status: **DRAFT**. Both developers must review and approve before M1 code is
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

**heartbeat**
```json
{
  "ts": "2026-10-07T10:00:00Z",
  "fw": "0.1.0",
  "uptime_s": 3600,
  "rssi": -61,
  "temp_c": 31.4,
  "voltage_v": 11.9,
  "current_a": 0.82,
  "mode": "AUTOMATIC"
}
```

**state**
```json
{ "ts": "...", "mode": "MANUAL", "fan": true, "uv": true, "co2": false, "camera": "ok" }
```

**command**
```json
{ "command_id": "c1b9…uuid", "type": "SET_FAN", "params": { "on": false }, "issued_at": "...", "expires_at": "..." }
```
Initial types: `SET_MODE`, `SET_FAN`, `SET_UV`, `SET_CO2`, `CAPTURE_IMAGE`, `SYNC_SCHEDULE`, `REBOOT`.

**ack**
```json
{ "command_id": "c1b9…", "result": "OK", "actual_state": { "fan": false }, "error": null, "ts": "..." }
```
`result`: `OK` | `FAILED` | `REJECTED` (invalid or expired) | `DUPLICATE` (already applied; returns the same state).

## Open questions

1. Heartbeat interval (30 s?) and offline timeout (3 missed beats?).
2. Device credentials: username/password per device vs. X.509 client certificates.
3. Does the device report fan health from tachometer/current draw, or only commanded state?
4. Image upload auth: same device credential, or short-lived upload token requested over MQTT?
