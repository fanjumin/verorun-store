# IoT Hub · 物联网接入中枢

`iot_hub` — v1.0.0 · VeroRun Official Plugin · `tools` · free

> **English:** An IoT device access hub for VeroRun. It ships a **built-in
> lightweight MQTT 3.1.1 broker** (zero external dependencies) plus an HTTP device
> API, telemetry storage, command dispatch and threshold alerting — all in a
> dedicated `iot_hub` PostgreSQL schema.
>
> **中文：** VeroRun 平台的物联网设备接入中枢。内置**轻量 MQTT 3.1.1 Broker**
> （激活即监听端口，用户侧零安装、零外部依赖）以及 HTTP 设备 API、遥测存储、
> 指令下发与阈值告警，数据全部存放在独立的 `iot_hub` PostgreSQL schema 中。

---

## Table of Contents · 目录

1. [Overview · 概述](#overview--概述)
2. [Features · 核心特性](#features--核心特性)
3. [Architecture · 架构与目录](#architecture--架构与目录)
4. [Database Schema · 数据库结构](#database-schema--数据库结构)
5. [Installation & Enablement · 安装与启用](#installation--enablement--安装与启用)
6. [Built-in MQTT Broker · 内置 MQTT Broker](#built-in-mqtt-broker--内置-mqtt-broker)
7. [Configuration · 配置项](#configuration--配置项)
8. [Device Onboarding · 设备接入](#device-onboarding--设备接入)
9. [Admin Dashboard · 管理界面](#admin-dashboard--管理界面)
10. [API Reference · API 文档](#api-reference--api-文档)
11. [MQTT Protocol Guide · MQTT 协议指南](#mqtt-protocol-guide--mqtt-协议指南)
12. [Agent Integration · Agent 集成](#agent-integration--agent-集成)
13. [Known Limitations · 已知限制](#known-limitations--已知限制)
14. [Troubleshooting · 故障排查](#troubleshooting--故障排查)
15. [Development & Tests · 开发与测试](#development--tests--开发与测试)

---

## Overview · 概述

**English**

`iot_hub` lets physical devices join the VeroRun platform over **MQTT 3.1.1**
(no broker to install — the plugin embeds one) or a simple **HTTP REST API**.
Devices are isolated per owner, authenticated with HMAC-signed requests or
short-lived tokens, report telemetry into time-series rows, receive downlink
commands, and are monitored by threshold alert rules.

**中文**

`iot_hub` 让真实设备通过 **MQTT 3.1.1**（无需安装 Broker，插件内置）或简单的
**HTTP REST API** 接入 VeroRun 平台。设备按属主隔离，采用 HMAC 签名或短期令牌
鉴权，遥测数据按时序入库，可接收下行指令，并受阈值告警规则监控。

---

## Features · 核心特性

| Feature · 特性 | English | 中文 |
|---|---|---|
| Built-in Broker · 内置 Broker | Pure-Python MQTT 3.1.1 broker, port 1883, zero external dependencies. | 纯 Python MQTT 3.1.1 Broker，端口 1883，零外部依赖。 |
| External Bridge · 外部桥接 | Connect to a remote broker (TLS supported) as a client when `broker_mode=external`. | `broker_mode=external` 时以客户端身份连接远程 Broker（支持 TLS）。 |
| Dual Channel · 双通道 | MQTT + HTTP REST both map to the same device identity. | MQTT 与 HTTP REST 共用同一套设备身份。 |
| HMAC Security · HMAC 安全 | Secret stored only as HMAC digest; per-request signature with a 5-minute replay window. | 密钥仅存 HMAC 摘要；请求级签名带 5 分钟重放窗口。 |
| Telemetry · 遥测 | JSON payloads with dotted-path metric extraction and series queries. | JSON 遥测载荷，支持点号路径指标提取与趋势序列查询。 |
| Commands · 指令 | queued → sent → delivered → acked state machine with TTL expiry. | queued → sent → delivered → acked 状态机，带 TTL 过期。 |
| Alerting · 告警 | Device-scoped and global threshold rules (`> >= < <= == !=`). | 设备级与全局阈值规则（`> >= < <= == !=`）。 |
| Retention · 保留策略 | Scheduled cleanup of telemetry older than `telemetry_retention_days`. | 定时清理超过保留期的遥测数据。 |

---

## Architecture · 架构与目录

```
plugins/iot_hub/
├── __init__.py                     # Plugin entry: lifecycle + self-owned maintenance thread
├── plugin.json                     # Manifest (menu, config, permissions, agent)
├── models.py                       # DB access, migrations, drop_schema
├── routes.py                       # Admin REST API (JWT admin-gated, /admin/iot)
├── public_routes.py                # Device REST API (HMAC/token, /api/iot/v1)
├── agents/
│   └── device_ops_analyst_prompt.md  # Sub-agent prompt (enabled_by_default=false)
├── services/
│   ├── broker.py                   # MQTT codec + route_message + MqttBroker (builtin)
│   ├── mqtt_bridge.py              # MqttBridge (external mode client)
│   ├── security.py                 # HMAC digest/signature, device tokens
│   ├── device_manager.py           # Device CRUD, online/offline, secret regeneration
│   ├── telemetry_store.py          # Ingest, query, series, retention
│   ├── command_dispatcher.py       # Command state machine, poll, sweep delivery
│   └── alerting.py                 # Threshold rule evaluation
├── migrations/v1.0.0_init.sql      # Initial schema (idempotent, schema_version tracked)
├── templates/iot_admin.html        # Admin SPA partial (window.l_iot_admin)
├── i18n/                           # en.yml / zh-CN.yml
└── tests/                          # MQTT codec / security / alerting unit tests
```

### Runtime model · 运行模型

The broker/bridge is **process-scoped**: in the gunicorn multi-worker admin service,
the first worker that binds port 1883 owns the broker; other workers log and skip.
Cross-worker command delivery is handled by the broker's own `_sweep_delivery()` loop
(every 2 s), so queued commands reach online devices regardless of which worker
received the admin request. A plugin-owned maintenance thread also runs the offline
check, command expiry and retention cleanup independently of the framework's
`register_jobs()` contract (which is declared but not auto-scheduled by the framework).

---

## Database Schema · 数据库结构

All tables live in the dedicated `iot_hub` schema; migrations are idempotent and
tracked in `schema_version`.

| Table | Purpose · 用途 |
|---|---|
| `devices` | `device_key` unique, `secret_hash`, protocol, status, `last_seen_at`, `metadata` jsonb, owner scoping. |
| `telemetry` | BIGSERIAL rows of `(device_id, topic, payload jsonb, ts, received_at)`, indexed `(device_id, ts DESC)`. |
| `commands` | Command state machine with `expires_at` TTL. |
| `alert_rules` | Threshold rules; `device_id NULL` = global rule. |
| `alerts` | Triggered alerts with `resolved_at`. |
| `schema_version` | Migration bookkeeping. |

---

## Installation & Enablement · 安装与启用

- **Requirements:** VeroRun ≥ 0.59.3, PostgreSQL. No Python packages required.
- **Install:** Admin → Plugins → Install → IoT Hub. On install, `migrate_all()` creates
  the schema; `seed_plugin_translations()` seeds i18n.
- **Enable:** Enabling starts the MQTT runtime (builtin broker listens on `mqtt_port`,
  default 1883) and the maintenance thread. On uninstall, `drop_schema()` removes only
  plugin-owned tables.

```bash
# Example (inside a VeroRun Python shell)
from plugin_manager import get_manager
m = get_manager()
m.install_plugin('iot_hub')
m.enable_plugin('iot_hub')
```

> **Security note:** when using the builtin broker, open TCP port `1883` in the server
> firewall only if devices connect from the internet. For local-only device fleets, set
> `mqtt_bind_host=127.0.0.1` and use the HTTP API instead.

---

## Built-in MQTT Broker · 内置 MQTT Broker

This plugin **does not require an external MQTT broker** (no mosquitto installation,
no systemd unit, no firewall surprises beyond one port). When `broker_mode=builtin`:

1. Enabling the plugin starts a daemon thread owning a listening socket on
   `mqtt_bind_host:mqtt_port` (default `0.0.0.0:1883`).
2. Devices connect with MQTT 3.1.1 and authenticate using
   **username = `device_key`, password = raw secret** (validated against the stored
   HMAC digest).
3. The broker routes three uplink topics and publishes downlink commands; it is
   auto-healing (`ensure_running()` restarts it if the thread dies).

**Alternative — external broker:** set `broker_mode=external`, `mqtt_host`,
`mqtt_username`, `mqtt_password` (and `mqtt_tls=true` for 8883). The plugin then acts
as an MQTT client, subscribing to `iot/+/telemetry|status|cmd/ack`. Use this when you
already operate a production broker such as EMQX / Mosquitto / AWS IoT Core.

---

## Configuration · 配置项

Managed in Admin → Plugins → IoT Hub → Settings (also in `plugin.json` `config`).

| Key | Default | Description |
|---|---|---|
| `broker_mode` | `builtin` | `builtin` (embedded broker) or `external` (remote broker client). |
| `mqtt_bind_host` | `0.0.0.0` | Address the builtin broker binds to. |
| `mqtt_port` | `1883` | Builtin broker port, or external broker port. |
| `mqtt_host` | `""` | External broker host (external mode only). |
| `mqtt_username` / `mqtt_password` | `""` | External broker credentials (external mode only). |
| `mqtt_tls` | `false` | Use TLS on 8883 (external mode). |
| `mqtt_topic_prefix` | `iot` | Topic prefix, e.g. `iot/<device_key>/telemetry`. |
| `telemetry_retention_days` | `30` | Retention window; cleaned daily at 03:30. |
| `offline_timeout_sec` | `300` | Device offline threshold. |
| `telemetry_max_payload_kb` | `64` | Max telemetry JSON payload size. |

---

## Device Onboarding · 设备接入

1. **Create the device** in Admin → Tools → IoT Hub → Devices → *Add Device*.
   The response shows a one-time `secret` (stored only as an HMAC digest).
2. **Connect over MQTT** (builtin mode):

   ```
   host     = <your-domain>
   port     = 1883
   username = <device_key>
   password = <device_secret>
   ```

3. **Or use the HTTP API** with the device key + secret (HMAC headers, see below).
4. Optionally **regenerate the secret** anytime; old signatures/tokens immediately fail.

---

## Admin Dashboard · 管理界面

Menu **Tools → IoT Hub** opens `templates/iot_admin.html` (`window.l_iot_admin`),
showing:

- **Stats:** Total Devices / Online / Telemetry 24h / Active Alerts.
- **Devices:** CRUD, one-time secret, telemetry table, ECharts trend series, command
  sending with a state panel, status badge.
- **Alerts & Rules:** active alerts, create/edit/delete threshold rules (device-scoped
  or global).

---

## API Reference · API 文档

### Admin API (Bearer JWT with `is_admin: true`)

Base prefix: `/admin/iot`

| Method | Path | Description |
|---|---|---|
| GET | `/admin/iot` | Dashboard stats. |
| GET | `/admin/iot/devices` | List devices (`limit`, `offset`, `status`, `q`). |
| POST | `/admin/iot/devices` | Create device → `{device, secret}` (one-time). |
| GET | `/admin/iot/devices/<id>` | Get device. |
| PUT | `/admin/iot/devices/<id>` | Update device (name/model/protocol/metadata). |
| DELETE | `/admin/iot/devices/<id>` | Delete device (cascades telemetry/commands/alerts). |
| POST | `/admin/iot/devices/<id>/secret` | Regenerate secret → `{secret}`. |
| GET | `/admin/iot/devices/<id>/telemetry` | List telemetry (`from`, `to`, `limit`, `offset`). |
| GET | `/admin/iot/devices/<id>/series?metric=temp.c` | Dotted-path trend series. |
| POST | `/admin/iot/devices/<id>/commands` | Send command `{command, payload, ttl_sec}` → `{command_id, status}`. |
| GET | `/admin/iot/commands` | List commands (`device_id`, `status`, `limit`, `offset`). |
| GET | `/admin/iot/alerts` | List alerts (`device_id`, `limit`, `offset`). |
| GET | `/admin/iot/alert-rules` | List rules. |
| POST | `/admin/iot/alert-rules` | Create rule `{metric_key, operator, threshold, severity, device_id, enabled}`. |
| PUT | `/admin/iot/alert-rules/<id>` | Update rule. |
| DELETE | `/admin/iot/alert-rules/<id>` | Delete rule. |
| GET | `/admin/iot/mqtt/status` | Broker/bridge runtime status (mode, bound/connected, clients). |

### Device API (HMAC headers or Bearer token)

Base prefix: `/api/iot/v1`

Authentication options:

- **HMAC signature** — headers `X-Device-Key`, `X-Timestamp` (epoch seconds),
  `X-Signature` = `hex(HMAC(secret_hash, timestamp))` where
  `secret_hash = hex(HMAC(secret, "iot_hub:v1"))`. Replay window = 300 s.
- **Bearer token** — obtain via `POST /auth`, then
  `Authorization: Bearer <token>`.

| Method | Path | Description |
|---|---|---|
| POST | `/api/iot/v1/auth` | `{ttl}` → `{token, expires_in}`. |
| POST | `/api/iot/v1/telemetry` | Report one reading (`{ts?, payload}` or flat payload). |
| POST | `/api/iot/v1/telemetry/batch` | `{items: [{ts, payload}]}` → `{stored}`. |
| POST | `/api/iot/v1/status` | `{online: true/false}` heartbeat. |
| GET | `/api/iot/v1/commands/poll` | Poll queued commands (HTTP-mode devices). |
| POST | `/api/iot/v1/commands/<id>/ack` | Acknowledge a delivered command. |

---

## MQTT Protocol Guide · MQTT 协议指南

Only MQTT 3.1.1 (protocol level 4) is supported. Implemented packets:
CONNECT/CONNACK, PUBLISH (QoS 0/1)/PUBACK, SUBSCRIBE/SUBACK,
UNSUBSCRIBE/UNSUBACK, PINGREQ/PINGRESP, DISCONNECT, plus Last Will.

Uplink topics (`<prefix>` defaults to `iot`):

| Topic | Payload | Effect |
|---|---|---|
| `<prefix>/<device_key>/telemetry` | JSON `{metric: value, ...}` | Store telemetry, touch `last_seen`, evaluate alert rules. |
| `<prefix>/<device_key>/status` | `{"online": false}` | Mark offline; anything else marks online. |
| `<prefix>/<device_key>/cmd/ack` | `{"command_id": "<uuid>"}` | Acknowledge a command. |

Downlink topic: `<prefix>/<device_key>/cmd` with payload
`{"command": "<name>", "payload": {...}, "command_id": "<uuid>"}`.
Publish it with QoS ≥ 1; the device should respond via `cmd/ack` with the same
`command_id`.

---

## Agent Integration · Agent 集成

`plugin.json` declares one sub-agent **Device Ops Analyst**
(`iot_hub_device_ops_analyst`) with capabilities
`device.telemetry.query / device.status.query / device.command.send /
device.alert.list`, reading `agents/device_ops_analyst_prompt.md`.
It is `enabled_by_default=false` — an explicit opt-in is required, so enabling the
plugin never changes agent behavior on its own.

---

## Known Limitations · 已知限制

- **Single-process broker:** the builtin broker runs in one gunicorn worker. If that
  worker restarts, the broker restarts with it (self-healing); a brief reconnect
  window is expected.
- **No retained messages / no wildcard fan-out:** the broker is a minimal embedded
  implementation. Use `broker_mode=external` for production-grade broker features.
- **v1.0 alerting has no cooldown:** every matching reading inserts an alert row;
  deduplication/notification channels are future work.
- **HTTP devices poll commands** (`/commands/poll`) — real-time push requires MQTT.

---

## Troubleshooting · 故障排查

| Symptom · 现象 | Cause · 原因 | Fix · 处理 |
|---|---|---|
| Broker not listening after enable | Port already in use / another worker owns it | Check `mqtt/status` (bound + clients); ensure only one process binds, or change `mqtt_port`. |
| Device can't connect (CONNACK 4) | Wrong username/password | Username must be `device_key`, password the raw `secret`; regenerate if lost. |
| 401 `invalid signature` on HTTP API | Clock skew / stale secret | X-Timestamp within 300 s of server time; regenerate and re-issue tokens. |
| Command stays `queued` | Device offline or not subscribed | Device must be online (MQTT or poll); commands expire via TTL. |
| Telemetry rejected 413 | Payload exceeds `telemetry_max_payload_kb` | Raise the limit or shrink the payload. |
| 401 on admin API | JWT lacks `is_admin` | Log in as an admin; pass the token via `Authorization: Bearer` or cookie. |
| Port 1883 unreachable from device | Firewall | Open TCP 1883, or set `mqtt_bind_host=127.0.0.1` and use the HTTP API. |

---

## Development & Tests · 开发与测试

```bash
# Unit tests (no database required)
python -m unittest plugins.iot_hub.tests.test_mqtt_codec -v
python -m unittest plugins.iot_hub.tests.test_security -v
python -m unittest plugins.iot_hub.tests.test_alerting -v
```

Covered scenarios:

- MQTT codec round-trips (CONNECT with credentials/will, PUBLISH QoS 0/1 incl. Chinese
  and binary payloads, SUBSCRIBE/UNSUBSCRIBE, PINGREQ/RESP, CONNACK codes).
- HMAC digest determinism, signature verification with tamper/replay/clock-skew
  rejection, device-token issuance/expiry/tamper, key extraction.
- Threshold alert evaluation against sqlite-backed rules: device-scoped, global,
  disabled, unknown operator, non-numeric paths.

---

*This README reflects plugin v1.0.0. Generated from the actual source; if any behavior
differs from this document, the source is authoritative. · 本文档对应插件 v1.0.0，
以实际源码为准。*
