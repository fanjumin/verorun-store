# Analytics (`analytics`)

> Server-side cookieless analytics middleware and dashboard. PV/UV, realtime
> online, geo & device insights, event tracking, funnels, alerts, privacy
> config, offline MaxMind/ip2region databases, ECharts visualization.

- **Identifier:** `analytics`
- **Version:** 1.11.3
- **Category:** system
- **Menu group:** Monitoring & Data → Analytics
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `content`
- **Python deps:** `user-agents`

---

## Core Features

### 1. Server-side collection (cookieless)
- Middleware writes one `analytics_logs` row per request — no frontend JS /
  cookie needed.
- `sample_rate` (default 1.0 = full).
- Parses device / OS / browser via `user-agents`.

### 2. 11 tables, multi-granularity aggregation
- **Raw**: `analytics_logs` (per request), `analytics_visitor_sessions`
  (sessions), `analytics_events` (custom events).
- **Aggregates**: `analytics_hourly_stats`, `analytics_daily_stats`,
  `analytics_page_stats`, `analytics_source_stats`,
  `analytics_geo_stats`, `analytics_device_stats`.
- **Config**: `analytics_alerts`, `analytics_privacy_config`.

### 3. Realtime & trend API (17 `/api/v1/*` endpoints)
- `POST /api/v1/log`, `POST /api/v1/event`
- `GET /api/v1/realtime`, `trend`, `hourly`
- `GET /api/v1/pages`, `sources`, `devices`
- `GET /api/v1/geo`, `geo/cities`, `geo/china-cities`, `geo/market`
- `GET /api/v1/events`, `overview`
- `GET /api/v1/stats`, `export`, `report`, `report/text`
- `POST /api/v1/cleanup`

### 4. Alerts
- `GET/POST /api/v1/alerts`, `PUT/DELETE /api/v1/alerts/<id>` — rules
  for PV spikes, error rates, etc.

### 5. GeoIP (offline)
- Ships `data/GeoLite2-City.mmdb` (62 MB) and `data/ip2region_v4.xdb`
  (10 MB).
- 5 GeoIP config endpoints under `/settings/geoip/`.
- Auto-update GeoLite2 via MaxMind Account ID + License Key.
- Static geo data: `static/geo/{us,in,jp,br}.json`, `world.json`,
  `china.json` for ECharts maps.

### 6. Privacy config
- `GET/PUT /api/v1/privacy` — control IP / UA / geo collection.

### 7. Dashboard
- Single page `templates/analytics.html` (30 KB) +
  `analytics-dashboard.js` (89 KB).
- Dashboard stats: today PV, today UV, online now, 30-day PV.

### 8. 3 hooks
- `analytics/track_event`, `analytics/get_realtime`,
  `analytics/get_trend`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `analytics`, **11 tables** (see above) |
| Blueprint | `/admin/analytics` (`routes.py`, 30 KB) |
| HTTP endpoints | **37** (collection, query, alerts, GeoIP config, privacy, export) |
| Dashboard | `templates/analytics.html` (30 KB) + `analytics-dashboard.js` (89 KB) |
| GeoIP DB | `GeoLite2-City.mmdb` (62 MB) + `ip2region_v4.xdb` (10 MB) |
| Python deps | `user-agents` |
| Permissions | `api:read`, `api:write`, `network:request`, `admin:access` |
| Capabilities | `analytics.track`, `analytics.report`, `analytics.geo`, `analytics.funnel` |
| Dashboard stats | `today_pv`, `today_uv`, `online_now`, `total_pv_30d` |

### Config (5 items)

| Key | Default | Purpose |
|---|---|---|
| `sample_rate` | 1.0 | Request sampling (0–1) |
| `geoip_enabled` | true | Enable GeoIP resolution |
| `service_name` | admin | Log service label |
| `maxmind_account_id` | empty | MaxMind Account ID for GeoLite2 |
| `maxmind_license_key` | empty | MaxMind License Key |

---

## Five-minute quick start

1. Enable — auto-creates 11 tables.
2. Open **Monitoring & Data → Analytics**.
3. Dashboard shows today PV/UV and online now.
4. (Optional) Configure MaxMind key under Settings for auto geo updates.
5. (Optional) Add an alert rule for PV spikes.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / integrator | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
