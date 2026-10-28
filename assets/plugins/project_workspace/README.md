# Project Workspace (`project_workspace`)

> AI-powered project workspace. Project isolation, document RAG (PDF /
> DOCX / PPTX / XLSX / CSV / TXT / MD), semantic search, QA with source
> citations, and a research assistant.

- **Identifier:** `project_workspace`
- **Version:** 1.6.5
- **Category:** ai_agent
- **Menu group:** System → Project Workspace
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `content`
- **Recommends:** memory_engine, chatbot, analytics
- **Python deps:** python-docx, pdfplumber, python-pptx

---

## Core Features

### 1. Project isolation
- `GET/POST /projects`, `GET /projects/<owner_type>`.
- `DELETE /projects/<id>`.
- `max_projects_per_user=50`, `max_documents_per_project=5000`.

### 2. Project members
- `GET/POST /projects/<id>/members`.

### 3. Document upload & RAG (`services/doc_processor.py`, 14.9 KB)
- `POST /documents`.
- Allowed: pdf / docx / txt / md / pptx / xlsx / csv.
- `max_file_size_mb=50`.
- Chunking: `chunk_size=1000`, `chunk_overlap=200`.
- Embedding: `text-embedding-3-small`.
- `DELETE /documents/<id>`, `GET /documents/<id>/status`.

### 4. Semantic search (`services/retriever.py`, 10 KB)
- `GET /search`.
- `semantic_search_top_k=10`.
- Optional `rerank_model`.
- `enable_cross_project_search=false` (project isolation by default).

### 5. QA with citations
- `GET /qa`, `GET /citations`, `GET /qa/history`, `POST /qa/feedback`.

### 6. AI assistants
- `POST /agent/summarize` — document summary.
- `POST /agent/compare` — document compare.
- `services/researcher.py` (4.3 KB) — research assistant.

### 7. Hooks
- Provided: `project_workspace/search`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `project_workspace` (migration v1.0.0_init.sql, 7.4 KB) |
| Blueprint | `/admin/project-workspace` (`routes.py`, 25 KB) |
| HTTP endpoints | **19** |
| Doc processor | `services/doc_processor.py` (14.9 KB) |
| Retriever | `services/retriever.py` (10 KB) |
| Researcher | `services/researcher.py` (4.3 KB) |
| Auth | `auth.py` (7.8 KB) |
| Admin UI | `templates/project_workspace_admin.html` (18 KB) |
| Python deps | python-docx, pdfplumber, python-pptx |
| Permissions | api:read/write, user:profile, network:request |
| Capabilities | document.summarize, document.compare, qa.with_sources, content.analyze |
| Hook provided | project_workspace/search |
| Dashboard | total_projects / total_documents / queries_24h / avg_response_time |

### Key config

| Key | Default | Purpose |
|---|---|---|
| `max_file_size_mb` | 50 | Upload size cap (1–200) |
| `chunk_size` | 1000 | Chunk size (200–5000) |
| `semantic_search_top_k` | 10 | Semantic search top-k |
| `embedding_model` | text-embedding-3-small | Embedding model |
| `storage_dir` | plugins/project_workspace/data/ | File storage dir (under vault backup convention) |
| `rerank_model` | empty | Optional rerank model |
| `retention_days` | 730 | Retention (2 years) |
| `enable_cross_project_search` | false | Cross-project search |

---

## Five-minute quick start

1. Enable — auto-creates tables.
2. Open **System → Project Workspace**.
3. Create project → add members → upload documents.
4. Wait for chunking / embedding.
5. Search / QA inside project with citations.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Knowledge workers / team leads | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Users | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
