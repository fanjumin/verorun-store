# ROS Bridge (`ros_bridge`)

> ROS 2 fleet gateway. Connect robots via rosbridge (WebSocket), ingest
> telemetry, dispatch allowlisted commands / missions, and feed the
> evolution flywheel.

- **Identifier:** `ros_bridge`
- **Version:** 1.0.1
- **Category:** tools
- **Menu group:** Tools → ROS Bridge
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, minipro, edge
- **Agent role:** `ops`
- **Python deps:** `websockets`
- **MCP Server:** `plugins/ros_bridge/tools/mcp_server.py` (stdio)

---

## Core Features

### 1. rosbridge WebSocket client (`client.py`, 6.7 KB)
- Connects ROS 2 fleet.
- `reconnect_backoff_max_sec=30`.

### 2. Telemetry ingestion
- `telemetry_throttle_ms=1000`.
- `telemetry_store_min_interval_sec=5`.
- `telemetry_retention_days=30`.
- `gateway.py` (6.7 KB).

### 3. Allowlisted commands
- `command_allowlist=[]` — only allowlisted commands dispatchable.
- `approval_timeout_sec=600`.
- `ros.before_command` hook for pre-dispatch interception.

### 4. Missions
- `robot.mission.list` / `robot.mission.cancel` capability.
- `missions_24h` dashboard.

### 5. Heartbeat
- `heartbeat_interval_sec=60`.

### 6. Graph cache
- `graph_cache_ttl_sec=300` (ROS TF graph).

### 7. Evolution flywheel
- `curation_push_enabled=false` — push robot performance to cogevolution.
- Listens to `cogevolution.owner_allowed`, `cogevolution.improvement.outcome`.

### 8. MCP
- `tools/mcp_server.py` (17.2 KB) — AI agent controls robots via MCP.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `ros_bridge` |
| Runtime | `runtime.py` (28 KB) |
| Blueprint | `/admin/ros` (`routes.py`, 24.1 KB) |
| Admin UI | `templates/ros_admin.html` (23.2 KB) |
| MCP | `tools/mcp_server.py` (17.2 KB) |
| Client | `client.py` (6.7 KB) |
| Python deps | `websockets` |
| Permissions | api:read/write, network:request, routes, scheduler |
| Capabilities | robot.status.query, robot.telemetry.query, robot.graph.query, robot.command.send, robot.mission.list, robot.mission.cancel |
| Hook provided | ros.before_command |
| Hooks listened | cogevolution.owner_allowed, cogevolution.improvement.outcome |
| Dashboard | robots_total / robots_online / missions_24h / commands_pending |

### Key config

| Key | Default | Purpose |
|---|---|---|
| `reconnect_backoff_max_sec` | 30 | Reconnect backoff cap |
| `telemetry_throttle_ms` | 1000 | Telemetry throttle |
| `telemetry_store_min_interval_sec` | 5 | Min store interval |
| `telemetry_retention_days` | 30 | Telemetry retention |
| `command_allowlist` | [] | Command allowlist |
| `approval_timeout_sec` | 600 | Approval timeout |
| `heartbeat_interval_sec` | 60 | Heartbeat interval |
| `curation_push_enabled` | false | Push to evolution flywheel |
| `graph_cache_ttl_sec` | 300 | TF graph cache TTL |

---

## Five-minute quick start

1. Enable — auto-creates tables.
2. Open **Tools → ROS Bridge**.
3. Configure rosbridge WebSocket URL.
4. Add command allowlist.
5. Fleet telemetry auto-ingested; AI agent dispatches via MCP.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Robotics engineers / ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
