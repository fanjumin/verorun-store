# IoT Hub (`iot_hub`)

> IoT device access hub. Built-in lightweight MQTT broker, HTTP device
> API, telemetry storage, downstream commands, and threshold alerting.

- **Identifier:** `iot_hub`
- **Version:** 1.2.1
- **Category:** tools
- **Menu group:** Tools → IoT Hub
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, minipro, edge
- **Agent role:** `ops`
- **Python deps:** none

---

## Core Features

### 1. Dual-mode MQTT broker
- `broker_mode=builtin`: plugin listens on `mqtt_port=1883`, devices
  connect directly.
- `broker_mode=external`: plugin connects as a client to a remote broker.
- `mqtt_bind_host=0.0.0.0` (production: bind 127.0.0.1 or enable TLS).
- `mqtt_tls=false` (production strongly recommended: TLS=8883, otherwise
  device_key / secret / telemetry travel in clear text).
- `mqtt_max_connections=200`.
- `mqtt_topic_prefix=iot`.

### 2. HTTP device API
- Devices authenticate with device_key / secret.
- `telemetry_max_payload_kb=64` (applies to both HTTP and MQTT;
  oversized packets are dropped and MQTT connection closed).

### 3. Telemetry storage & retention
- `telemetry_retention_days=30`.
- `offline_timeout_sec=300`.

### 4. Downstream commands
- `device.command.send` capability.
- `commands_retention_days=30` for acked / failed commands.

### 5. Threshold alerting
- `device.alert.list` capability.
- `alerts_retention_days=90` (resolved alerts age from resolution time).
- `alert_cooldown_sec=300` per rule per device (0 = off).

### 6. 4 dashboard stats
- `total_devices`, `online_devices`, `telemetry_24h`, `alerts_active`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `iot_hub` (schema_version metadata) |
| Access | Built-in MQTT broker or external broker + HTTP device API |
| Capabilities | device.telemetry.query, device.status.query, device.command.send, device.alert.list |
| Site capability | `iot` |
| Permissions | api:read/write, network:request, routes, scheduler, events, dag |
| Dashboard | total_devices / online / telemetry_24h / alerts_active |

### Key config

| Key | Default | Purpose |
|---|---|---|
| `broker_mode` | builtin | builtin / external |
| `mqtt_bind_host` | 0.0.0.0 | Built-in broker bind address |
| `mqtt_port` | 1883 | MQTT port |
| `mqtt_host` | empty | External broker host |
| `mqtt_username` / `mqtt_password` | empty | MQTT credentials |
| `mqtt_tls` | false | TLS (production recommended) |
| `mqtt_max_connections` | 200 | Built-in broker connection cap |
| `telemetry_retention_days` | 30 | Telemetry retention |
| `alerts_retention_days` | 90 | Alert retention |
| `commands_retention_days` | 30 | Command retention |
| `offline_timeout_sec` | 300 | Offline threshold |
| `telemetry_max_payload_kb` | 64 | Payload cap |
| `alert_cooldown_sec` | 300 | Alert cooldown |

---

## Five-minute quick start

1. Enable — auto-creates tables.
2. Open **Tools → IoT Hub**.
3. Pick builtin / external; enable TLS in production.
4. Register a device for device_key / secret.
5. Devices report telemetry via MQTT or HTTP; thresholds trigger alerts.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | IoT engineers / ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
