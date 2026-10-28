# VeroScholar (`veroscholar`)

> Full-cycle research AI assistant: topic, literature search (arXiv /
> Semantic Scholar / OpenAlex / CN), tags, notes, review generation,
> citation verification, writing aid, and a Discovery hypothesis engine.
> **Research edition only.**

- **Identifier:** `veroscholar`
- **Version:** 1.11.4
- **Category:** ai_agent
- **Menu group:** AI & Content → VeroScholar
- **Min app version:** 0.10.0
- **Compatible editions:** **research only**
- **Agent role:** `veroscholar`
- **Recommends:** project_workspace
- **Python deps:** feedparser (arXiv RSS), pypdf

---

## Core Features

### 1. Literature search (4 sources)
- arXiv (`arxiv_enabled=true`).
- Semantic Scholar (`semantic_enabled=true`).
- OpenAlex (`openalex_enabled=true`).
- CN literature (OpenAlex CN works, `zh_enabled=true`).
- `POST /api/v1/search`, `default_search_limit=30`.

### 2. Paper library
- `GET /api/v1/papers`, `GET /api/v1/papers/<id>`.
- `PATCH /api/v1/papers/<id>/status`.
- `GET/POST /api/v1/papers/<id>/tags`, `DELETE .../tags/<id>`.
- `GET /api/v1/tags`.
- `GET /api/v1/papers/<id>/export`.

### 3. Fulltext / translate / chat
- `POST/GET /api/v1/papers/<id>/fulltext`.
- `POST /api/v1/papers/<id>/translate`.
- `POST /api/v1/papers/<id>/chat`.
- `GET /api/v1/papers/<id>/related`.

### 4. Annotations
- `POST /api/v1/papers/<id>/annotations`.

### 5. Research projects
- `GET/POST /api/v1/projects`, `GET /api/v1/projects/<id>`,
  `POST /api/v1/projects/<id>/papers`.

### 6. Review / citation verify
- `GET/POST /api/v1/reviews`, `GET /api/v1/reviews/<id>`,
  `GET /api/v1/reviews/<id>/verify`.

### 7. Workflow (`workflow.py`, 24.4 KB)
- `POST /api/v1/workflow/run`,
  `GET /api/v1/workflow/instances/<id>`.

### 8. Discovery engine (`discovery/`, ~80 KB)
- `discovery_enabled=false` by default.
- `POST /api/v1/discovery/trigger`, `GET /api/v1/discovery/runs`,
  `GET .../runs/<id>/hypotheses`, `.../discussions`,
  `POST .../hypotheses/<id>/vote`, `GET /api/v1/discovery/stats`.
- `discovery_per_source=6`, `discovery_elo_rounds=3`,
  `hypothesis_model_tier=economy`.
- `discovery/nodes.py` (28.4 KB), `discovery/discuss.py` (15.1 KB).

### 9. KB sync (`services/kb_sync.py`, 18.7 KB)
- Push to project_workspace.

### 10. Hooks
- Provided: `veroscholar.search`, `veroscholar.annotate`.

---

## At a glance

| Area | Value |
|---|---|
| Blueprints | `/veroscholar` (`routes.py`, 36.1 KB) + `/discovery` (`routes_discovery.py`, 20 KB) |
| HTTP endpoints | **40+** |
| Discovery | `discovery/nodes.py` (28.4 KB) |
| Workflow | `workflow.py` (24.4 KB) |
| Models | `models.py` (27.6 KB) |
| KB sync | `services/kb_sync.py` (18.7 KB) |
| Search UI | `templates/search.html` (24.2 KB) |
| Python deps | feedparser, pypdf |
| Permissions | network:request, api:read/write |
| Capabilities | 23 (lit.* / citation.* / exp.* / writing.* / discovery.*) |
| Hooks provided | veroscholar.search, veroscholar.annotate |
| Dashboard | total_papers / total_notes / total_projects / searches_24h |

### Config (10 items)

| Key | Default | Purpose |
|---|---|---|
| `semantic_scholar_api_key` | empty | Rate-limit boost |
| `openalex_api_key` | empty | Rate-limit boost |
| `default_search_limit` | 30 | Search count |
| `arxiv_enabled` | true | arXiv |
| `semantic_enabled` | true | Semantic Scholar |
| `openalex_enabled` | true | OpenAlex |
| `zh_enabled` | true | CN works |
| `discovery_enabled` | false | Discovery engine |
| `discovery_per_source` | 6 | Evidence per source |
| `discovery_elo_rounds` | 3 | Elo rounds |
| `hypothesis_model_tier` | economy | Hypothesis model tier |

---

## Five-minute quick start

1. **Research edition** enable — install feedparser / pypdf.
2. Open **AI & Content → VeroScholar**.
3. Search → library → tags.
4. Translate / chat / annotate.
5. Enable Discovery when needed.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Researchers / grad students | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Researchers | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
