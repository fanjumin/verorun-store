# Health Check (`health_check`)

> Automated system health monitoring. Runs a library of checkers on a
> schedule (CPU / memory / disk / DB / Redis / network), threshold
> alerting, trend analysis, AI root-cause analysis, AI auto-fix, and remote
> server snapshots.

- **Identifier:** `health_check`
- **Version:** 1.15.0
- **Category:** system
- **Menu group:** Monitoring & Data → Health Check
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `ops`
- **Python deps:** `redis`

---

## Core Features

### 1. Checker library (`checkers.py`, 88 KB)
- Built-in checkers: CPU / memory / disk / PostgreSQL / Redis / network /
  HTTP / cert / port.
- `discovery.py` (21.9 KB) — auto-discovery of available checkers.
- `POST /api/checkers/register` — third-party custom checker registration.

### 2. Scheduled checks
- `check_interval_minutes=5`.
- `POST /api/run` — manual trigger.
- `POST /api/internal/run` — internal scheduler callback.

### 3. Threshold alerting
- `cpu_threshold=90`, `mem_threshold=85`, `disk_threshold=85`.
- Channels: `alert_email`, `webhook_url`.
- Silence: `POST /api/alerts/silences`,
  `DELETE /api/alerts/silences/<id>`.
- Read: `POST /api/alerts/read`.

### 4. Trend analysis
- `health_trend` table for time-series.
- `GET /api/trend`, `GET /api/metrics`.

### 5. AI RCA + auto-fix
- `ai_fixer.py` (25.3 KB).
- `POST /api/ai-analyze` — root-cause analysis.
- `POST /api/ai-fix` — AI auto-fix (audited).
- `fix_audit_log` table.
- `POST /api/fix/rollback/<id>` — rollback.
- `GET /api/fix/audit` — audit list.

### 6. Remote server snapshots
- `remote_spec.py` (13.5 KB).
- `GET /api/remote/spec`, `POST /api/remote/ingest`,
  `GET /api/remote/servers`,
  `GET /api/remote/servers/<id>/snapshots`.
- `remote_snapshots` table.

### 7. Link scan
- `POST /api/links/scan`, `GET /api/links/report`.

### 8. Dashboard
- `health_score` (gauge), `health_passed`, `health_warnings`,
  `health_errors`, `unread_alerts`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `health_check`, **9 tables** |
| Blueprint | `/admin/health` (`routes.py`, 55.6 KB) |
| HTTP endpoints | **30** |
| Checker library | `checkers.py` (88 KB) |
| AI fixer | `ai_fixer.py` (25.3 KB) |
| Discovery | `discovery.py` (21.9 KB) |
| Remote spec | `remote_spec.py` (13.5 KB) |
| Admin UI | `templates/health.html` (69.6 KB) |
| Python deps | `redis` |
| Permissions | `api:read`, `api:write`, `network:request` |
| Capabilities | `health_analysis`, `fix_suggestion`, `root_cause_analysis` |
| Hooks provided | `health/run_check`, `health/get_status`, `health/get_trend` |
| Dashboard | health_score / passed / warnings / errors / unread_alerts |

### 9 tables

| Table | Purpose |
|---|---|
| `health_checks` | Checker definitions |
| `check_runs` | Per-run executions |
| `check_history` | Historical results |
| `alert_config` | Alert config |
| `alert_history` | Alert history |
| `alert_silences` | Silence rules |
| `health_trend` | Time-series trend |
| `fix_audit_log` | AI fix audit |
| `remote_snapshots` | Remote server snapshots |

### Key config

| Key | Default | Purpose |
|---|---|---|
| `alert_email` | empty | Alert email |
| `webhook_url` | empty | External alert webhook |
| `check_interval_minutes` | 5 | Check interval |
| `cpu_threshold` | 90 | CPU alert % |
| `mem_threshold` | 85 | Memory alert % |
| `disk_threshold` | 85 | Disk alert % |

---

## Five-minute quick start

1. Enable — auto-creates 9 tables.
2. Open **Monitoring & Data → Health Check**.
3. Set alert email / webhook.
4. Tune CPU / memory / disk thresholds.
5. Watch health_score; on anomaly run AI RCA / auto-fix.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops / SRE | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
