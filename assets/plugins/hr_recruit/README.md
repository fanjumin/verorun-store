# HR Recruit (`hr_recruit`)

> Compliance-first recruitment copilot. JD drafting + bias review, resume
> structuring, talent-pool vector retrieval, interview scheduling, and
> crypto-shred PII destruction based on retention policy. All AI decisions
> are fully auditable; advisory only, never replaces human judgment.

- **Identifier:** `hr_recruit`
- **Version:** 1.3.4
- **Category:** ai_agent
- **Menu group:** Operations → HR Recruit
- **Min app version:** 0.10.0
- **Compatible editions:** standard, enterprise
- **Agent role:** `office`
- **Python deps:** `pypdf>=4.0.0`, `python-docx>=1.1.0`, `cryptography>=42.0.0`
- **MCP Server:** `plugins/hr_recruit/tools/mcp_server.py` (stdio)

---

## Core Features

### 1. JD drafting + signoff (`jd.py`, 10.9 KB)
- `POST /api/jobs/<job_id>/jd` — AI-draft JD.
- `GET /api/jobs/<job_id>/jd/versions` — version history.
- `POST /api/jobs/<job_id>/jd/<version>/signoff` — human signoff.
- Every version is independently logged.

### 2. Bias review (`bias.py`, 29.1 KB)
- `POST /api/bias/scan` — scan JD / resume for gender / age / region bias
  terms.
- `hr_bias_metric` table.
- `bias_scan_rate_limit=30` per operator per minute.

### 3. Resume parsing (`parsers/resume_parser.py`, 9.7 KB)
- `POST /api/resume/parse` — upload PDF / DOCX, extract structured fields.
- `pypdf` + `python-docx`.
- `max_upload_mb=10` hard cap.
- `resume_upload_rate_limit=10` per minute.
- Raw in `hr_resume_raw`; structured in `hr_candidate_profile`.

### 4. Talent-pool vector search
- `hr_talent_embedding` table.
- `POST /api/talent/search` — semantic search.

### 5. Interview scheduling (`scheduling.py`, 10.3 KB)
- `POST /api/schedule/propose` — propose slots.
- `POST /api/applications/<id>/schedule` — confirm.
- `work_timezone=Asia/Shanghai`.

### 6. Application funnel
- `POST /api/applications` — create.
- `GET /api/applications` — funnel list.
- `POST /api/applications/<id>/stage` — advance stage.
- `hr_decision_log` records every decision.

### 7. Crypto-shred PII destruction (`crypto.py`, 9.4 KB + `retention.py`, 11.3 KB)
- **KEK** (`hr_master_key`) wraps per-candidate DEK.
- `hr_candidate_dek` stores DEKs.
- `purge_mode=crypto_shred` (default): destroy DEK → all copies (incl.
  backups) permanently unreadable.
- `hard_delete`: remove live rows only.
- Retention: `resume_raw_days=180`, `candidate_pii_days=365`.
- `notify_before_days=7` pre-purge alert.
- `POST /api/retention/purge` batch, `purge_batch_max=50`.
- `hr_purge_receipt` table.

### 8. DAG nodes (`dag_nodes.py`, 7.7 KB)
- Recruitment flow hooks into DAG.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `hr_recruit`, **11 tables** |
| Blueprint | `/admin/hr-recruit` (`routes.py`, 35.3 KB) |
| HTTP endpoints | **27** |
| Core modules | bias.py (29 KB), retention.py (11 KB), jd.py (11 KB), scheduling.py (10 KB), crypto.py (9 KB), resume_parser.py (9.7 KB) |
| MCP | `tools/mcp_server.py` (13.5 KB) |
| Admin UI | `templates/hr_recruit.html` (17.3 KB) |
| Python deps | pypdf, python-docx, cryptography |
| Permissions | api:read/write, admin:access, routes, scheduler, dag, health, events |
| Capabilities | hr.jd_draft, hr.bias_review, hr.resume_parse, hr.talent_search, hr.interview_schedule, hr.retention_purge |
| Hooks provided | hr.jd.drafted, hr.resume.parsed, hr.application.stage_changed, hr.retention.purged |
| Hooks listened | plugin_enabled, plugin_disabled |
| Dashboard | open_jobs, candidates_total, pending_purge |

### 11 tables

| Table | Purpose |
|---|---|
| `hr_job` | Positions |
| `hr_job_description` | JD versions |
| `hr_candidate` | Candidate master |
| `hr_candidate_dek` | Per-candidate data key (DEK) |
| `hr_resume_raw` | Raw resume files + text |
| `hr_candidate_profile` | Structured profile |
| `hr_talent_embedding` | Talent vectors |
| `hr_application` | Application funnel |
| `hr_decision_log` | Decision audit |
| `hr_bias_metric` | Bias metrics |
| `hr_purge_receipt` | Destruction receipts |

### Key config

| Key | Default | Purpose |
|---|---|---|
| `hr_master_key` | **required** | KEK (loss → all PII permanently unreadable) |
| `llm_tier` | standard | high / standard / cheap / local (on-prem) |
| `work_timezone` | Asia/Shanghai | Work timezone |
| `resume_raw_days` | 180 | Raw resume retention |
| `candidate_pii_days` | 365 | Candidate PII retention |
| `max_upload_mb` | 10 | Upload size cap |
| `notify_before_days` | 7 | Pre-purge alert days |
| `purge_mode` | crypto_shred | crypto_shred / hard_delete |
| `purge_batch_max` | 50 | Batch purge cap |
| `bias_scan_rate_limit` | 30 | Bias scan RPM |
| `resume_upload_rate_limit` | 10 | Resume upload RPM |

---

## Five-minute quick start

1. Enable — **must set `hr_master_key`** (KEK) first.
2. Open **Operations → HR Recruit**.
3. Create job → AI-draft JD → bias scan → human signoff.
4. Candidate uploads resume → parsed → funnel.
5. On retention expiry, auto / manual crypto-shred PII, receipt issued.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | HR / recruiting ops | `README.md` |
| Technical whitepaper | Developers / compliance | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | HR / hiring managers | [`docs/user-guide.md`](docs/user-guide.md) |
| Compliance | Legal / compliance | [`docs/compliance.md`](docs/compliance.md) |
| Acceptance | QA | [`docs/acceptance.md`](docs/acceptance.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
