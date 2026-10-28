# AI Site Builder (`site_builder`)

> LLM-driven website builder: prompt templates per industry (corporate /
> e-commerce / landing), site tasks, unified design tokens, visual block
> editor, design candidates, style presets, and version history.

- **Identifier:** `site_builder`
- **Version:** 2.12.0
- **Category:** content
- **Menu group:** System → AI Site Builder
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, minipro
- **Agent role:** `builder`
- **Python deps:** none

---

## Core Features

### 1. Prompt templates
- `site_builder_prompts` table.
- `GET/POST /prompts`, `GET/PUT/DELETE /prompts/<id>`.
- `GET /prompts/<identifier>`.

### 2. Site tasks
- `site_builder_tasks` table.
- `POST /execute` — run task.
- `GET /tasks`, `GET /tasks/<id>`.

### 3. Engine (`engine.py`, 42.4 KB)
- LLM-driven page generation.

### 4. Block system (`block_schemas.py`, 29.5 KB)
- `GET /api/block-params`.
- `POST /api/draft/update-block`, `update-block-order`, `delete-block`,
  `add-block`, `update-tokens`, `upload-image`.

### 5. Design tokens
- `POST /api/draft/update-tokens`.
- `site_settings.tokens` capability.

### 6. Design candidates / style presets
- `POST /design-candidates`, `POST /design-candidates/apply`.
- `GET /style-presets`, `GET /style-presets/<id>`,
  `POST /style-presets/<id>/apply`.

### 7. Site analyzer (`site_analyzer/profiler.py`, 27.4 KB)
- `site_analyzer.analyze` capability.

### 8. Preview / publish
- `POST /preview`, `GET /preview-static/<filename>`,
  `GET /preview-site`.
- `POST /publish`, `GET /publish-checklist`.
- `POST /modify`, `GET /draft-data`.

### 9. Wizard
- `GET/POST /wizard/brief`.

### 10. Version history
- `GET /versions`, `GET /versions/<id>`, `POST /versions/<id>/restore`.

### 11. Page management
- `GET/POST /pages`, `PUT /pages/<id>`.
- `GET /page-summary/<page>`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `site_builder`, **2 main tables** |
| Blueprint | `/admin/site-builder` (`routes.py`, 72.2 KB) |
| HTTP endpoints | **35+** |
| Engine | `engine.py` (42.4 KB) |
| Block schemas | `block_schemas.py` (29.5 KB) |
| Analyzer | `site_analyzer/profiler.py` (27.4 KB) |
| Admin UI | `admin_site_builder.html` (155 KB) + `ai_site_preview.html` (31 KB) |
| Python deps | none |
| Permissions | admin:access, database:write, network:request, file:write |
| Capabilities | site_builder.build, site_builder.publish, site_settings.tokens, site_analyzer.analyze |
| Hooks provided | site_builder.build, site_builder.publish, site_settings.tokens, site_analyzer.analyze |
| Dashboard | total_tasks / completed_tasks / total_prompts / total_analyses |

### 2 main tables

| Table | Purpose |
|---|---|
| `site_builder_prompts` | Prompt templates |
| `site_builder_tasks` | Site tasks |

### Config (3 items)

| Key | Default | Purpose |
|---|---|---|
| `schema` | site_builder | Schema name |
| `internal_base_url_env` | MAIN_SITE_INTERNAL_URL | Internal site URL env |
| `internal_token_env` | INTERNAL_SERVICE_TOKEN | Internal service token env |

---

## Five-minute quick start

1. Enable — auto-creates tables.
2. Open **System → AI Site Builder**.
3. Run wizard/brief or pick a prompt template.
4. Tweak layout / design tokens / upload images in block editor.
5. Preview → publish.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Ops / marketing / site builders | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Users | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
