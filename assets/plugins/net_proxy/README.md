# Net Proxy (`net_proxy`)

> Unified outbound network proxy and egress governance for plugins. Manages
> proxy channels (HTTP / SOCKS5), per-domain routing rules, circuit breaker,
> connectivity probing, and request logging.

- **Identifier:** `net_proxy`
- **Version:** 1.0.0
- **Category:** tools
- **Menu group:** System → Net Proxy
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `ops`
- **Python deps:** none (optional `PySocks>=1.7.0` for SOCKS5)

---

## Core Features

### 1. Proxy channels (`channels.py`, 19 KB)
- HTTP / SOCKS5 channels.
- `GET/POST /admin/channels`, `PUT/DELETE /channels/<id>`.

### 2. Per-domain rules (`rules.py`, 23.8 KB)
- `GET/POST /admin/rules`, `PUT/DELETE /rules/<id>`.
- `default_policy=DIRECT` (DIRECT / PROXY / BLOCK).

### 3. Circuit breaker (`fuse.py`, 11.9 KB)
- `fuse_threshold=3` consecutive failures.
- `fuse_cooldown_minutes=10` auto-recover after cooldown.

### 4. Connectivity probing
- `probe_interval_minutes=5`.
- `probe_target`: empty = probe channel endpoint itself.
- `POST /channels/<id>/probe` manual probe.

### 5. Egress execution (`egress.py`, 17.3 KB)
- Plugins send outbound requests through this plugin, auto-routed by rules.
- `default_timeout_s=10`.

### 6. Request log
- `log_retention_days=30`.
- `GET /admin/logs`.

### 7. Hooks
- Provided: `net_proxy/get_status`, `net_proxy/get_channels`,
  `net_proxy/get_request_log`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `net_proxy` (dynamic tables) |
| Blueprint | `/admin/net-proxy` (`routes.py`, 18.9 KB) |
| HTTP endpoints | **11** |
| Channels | `channels.py` (19 KB) |
| Rules | `rules.py` (23.8 KB) |
| Fuse | `fuse.py` (11.9 KB) |
| Egress | `egress.py` (17.3 KB) |
| Admin UI | `templates/net_proxy.html` (23.4 KB) |
| Python deps | none (optional PySocks) |
| Permissions | api:read/write, network:request, routes, scheduler, health |
| Capabilities | net_proxy_status, net_proxy_channel_manage, net_proxy_egress_route |
| Hooks provided | net_proxy/get_status, net_proxy/get_channels, net_proxy/get_request_log |
| Dashboard | channels / requests_today / probe_healthy / probe_failed |

### Key config

| Key | Default | Purpose |
|---|---|---|
| `default_timeout_s` | 10 | Default timeout (1–120s) |
| `probe_interval_minutes` | 5 | Probe interval |
| `probe_target` | empty | Probe target URL |
| `fuse_threshold` | 3 | Failures before fuse |
| `fuse_cooldown_minutes` | 10 | Fuse cooldown |
| `log_retention_days` | 30 | Log retention |
| `default_policy` | DIRECT | DIRECT / PROXY / BLOCK |

---

## Five-minute quick start

1. Enable — auto-creates tables.
2. Open **System → Net Proxy**.
3. Add proxy channel (HTTP / SOCKS5).
4. Add per-domain rules (proxy / direct / block).
5. Plugin outbound requests auto-governed.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Ops / SRE | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
