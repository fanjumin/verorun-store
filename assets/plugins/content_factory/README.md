# Content Factory (`content_factory`)

> Multi-source content collection (RSS / web / CNINFO financial disclosures)
> → DashScope Qwen AI processing → review workflow → publishing → skill
> push / static page generation / knowledge-base push.

- **Identifier:** `content_factory`
- **Version:** 1.7.0
- **Category:** content
- **Menu group:** AI & Content → Content Factory
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `content`
- **Python deps:** none

---

## Core Features

### 1. Multi-source collectors (3)
- `services/base_collector.py` (4 KB) — collector base class.
- `services/collectors/web_collector.py` (3.9 KB) — web / RSS.
- `services/collectors/cninfo_collector.py` (5.9 KB) — CNINFO financial
  disclosures.
- On first enable, seeds preset finance sources (Wallstreetcn / 36kr /
  WSJ, see `preset_sources.py`), gated by `preset_finance_sources=true`.

### 2. AI processing (`services/ai_processor.py`, 9.9 KB)
- Calls DashScope Qwen (`dashscope_text_key`).
- `POST /ai-format` — rewrite title / body.
- `POST /ai-cover` — AI-generated cover image.
- `max_items_per_run=10` per run.

### 3. Review workflow
- Raw → `raw_contents`; processed → `processed_contents`.
- `skip_review=false` (default) requires manual `POST /review` before
  publish queue.
- `auto_publish=false` (default) does not auto-publish after AI.

### 4. Publishing & push
- `POST /publish` — publish to site.
- `POST /push-to-knowledge` — push to main KB.
- `POST /generate-static` — generate static page.
- `POST /push-skill` + `services/skill_pusher.py` (5.1 KB) — package
  content as installable skill for agents.
- `GET /api/v1/skills`, `GET /api/v1/skills/<skill_name>/download` — skill
  endpoints for external agents. **Disabled by default** (`SKILL_API_ENABLED=1`
  to enable) and, when enabled, they only expose `status='published'` assets.

### 5. Scheduling
- Listens to `cron.tick`.
- `POST /cron/tick` receives scheduler callback.
- `GET /schedules`, `GET /cron` inspect schedules.

### 6. 36 HTTP endpoints (grouped)
- **Source mgmt**: `GET/POST /sources`, `PUT/DELETE /sources/<id>`
- **Crawl**: `POST /crawl`, `POST /cache/clear`
- **Raw**: `GET /contents`, `DELETE /contents/<id>`
- **Process**: `POST /process`, `POST /ai-format`, `POST /ai-cover`
- **Processed**: `GET /processed`, `GET/PUT /processed/<id>`,
  `POST /processed/batch-delete`, `POST /processed/<id>/archive`,
  `POST /processed/archive-batch`
- **Review / publish**: `POST /review`, `POST /publish`,
  `POST /push-to-knowledge`, `POST /generate-static`
- **Tasks / stats**: `GET /tasks`, `GET /stats`
- **Skill push**: `POST /push-skill`, `GET /pushed-skills`,
  `DELETE /pushed-skills/<id>`
- **External-agent skill API** (default off, published-only): `GET /api/v1/skills`,
  `GET /api/v1/skills/<skill_name>/download`
- **Schedule**: `POST /cron/tick`, `GET /schedules`, `GET /cron`
- **Settings**: `GET/POST /settings`
- **Personalized feed**: `GET /feed/personalized`

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `content_factory`, **7 tables** |
| Blueprint | `/admin/content-factory` (`routes.py`, 40.6 KB) |
| HTTP endpoints | **36** |
| Admin UI | `templates/admin_contentfactory.html` (30.6 KB) |
| Python deps | none |
| Permissions | `api:read`, `api:write`, `network:request` |
| Capabilities | `content.process`, `content.publish`, `content.push_skill` |
| Hooks provided | `content_factory/collect`, `content_factory/process`, `content_factory/publish`, `content_factory/push_skill` |
| Hooks listened | `cron.tick` |
| Dashboard | `source_count`, `pending`, `processed`, `published`, `failed` |

### 7 tables

| Table | Purpose |
|---|---|
| `content_sources` | Sources (RSS / web / CNINFO) |
| `raw_contents` | Raw fetched items |
| `processed_contents` | AI-processed items |
| `content_tasks` | Task queue |
| `skill_pushes` | Skill push records |
| `agent_registry` | Agent registry |
| `schema_meta` | Schema metadata |

### Config (5 items)

| Key | Default | Purpose |
|---|---|---|
| `dashscope_text_key` | empty | DashScope Qwen API Key |
| `max_items_per_run` | 10 | Max items per run |
| `skip_review` | false | Skip manual review |
| `auto_publish` | false | Auto-publish after AI |
| `preset_finance_sources` | true | Seed finance RSS on first enable |

---

## Five-minute quick start

1. Enable — auto-creates 7 tables, seeds preset finance sources.
2. Open **AI & Content → Content Factory**.
3. Set `dashscope_text_key`.
4. Add a source → manually `POST /crawl`.
5. Review processed items → publish / push to KB / push as skill.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / integrator | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops / editors | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
