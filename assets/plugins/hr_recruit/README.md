# HR Recruit (hr_recruit)

Recruitment copilot — **AI advises, humans decide; every decision is auditable.**

> Version: v1.1.0 · Per plugin-standard-v1.8 / i18n-standard v1.0 / VeroRun v0.61.0
> Status: fully implemented skeleton + docs

---

## 1. Positioning & Boundaries

### 1.1 Positioning

For **mid-to-large enterprises running VeroRun on their own servers**, hr_recruit provides a pluggable recruitment capability layer: JD drafting, resume structuring, talent pool search, and interview scheduling are delegated to the Agent — while the decision to advance or reject a candidate, plus the full audit trail, stays with humans.

### 1.2 What it explicitly does NOT do

| Not doing | Reason |
|-----------|--------|
| Auto-reject decisions | PIPL Art.24 / GDPR Art.22 / EU AI Act all require substantial human involvement |
| Emotion recognition, personality inference | EU AI Act Annex III prohibits workplace emotion analysis |
| AI video interview auto-scoring | High-risk; requires additional compliance assessment (e.g. Illinois AIVIA written consent) |
| Replace ATS | First version is an enhancement layer alongside an existing ATS |

### 1.3 Differentiation

Not competing on "who screens more accurately" with Beisen/Moka, but on **data sovereignty + decision traceability**:

| Dimension | Beisen / Moka | hr_recruit |
|-----------|--------------|------------|
| Form | Full-stack ATS SaaS (closed) | Pluggable capability layer on customer's own server |
| Data location | Vendor cloud | **Customer server, fully on-prem capable** |
| Model | Vendor proprietary vertical model | UnifiedLLM gateway, switchable to `tier=local` |
| Auditability | Vendor reports | Decision log + prompt version + evidence citations, fully exportable |
| Extensibility | Vendor roadmap | Plugin + DAG + event bus, customer-orchestrated |

---

## 2. Capability Matrix & Rollout Order

| Stage | Risk | Automation form | Human involvement | Order |
|-------|------|-----------------|-------------------|-------|
| Interview scheduling | Low | Fully auto (cross-calendar intersection) | None | **① First** |
| JD generation + bias review | Low | 4-stage protocol generation | Human sign-off before publish | ② |
| Candidate status notifications | Low | Fully auto | None | ③ |
| Resume parsing & structuring | Medium | AI extraction + confidence labels | Low-confidence manual correction | ④ |
| Talent pool semantic search | Medium | RAG + evidence citations | Results are candidate sets, not conclusions | ⑤ |
| Retention purge | Medium | Cron + key destruction | Checklist review before execution | ⑥ |
| Match scoring & ranking | Medium | v0.2 | **Must be human-reviewed** | v0.2 |
| Auto-reject / emotion analysis | High/Prohibited | **Not doing** | — | — |

**Rollout order must follow risk order**: start with scheduling (where failure costs only a reschedule) to build team trust; scoring/ranking comes last.

**First version scope (confirmed): all six capabilities live** — `hr.jd_draft` / `hr.bias_review` / `hr.resume_parse` / `hr.talent_search` / `hr.interview_schedule` / `hr.retention_purge`. Candidate status notifications are a process detail carried by DAG `wait` + `hr.application.stage_changed` events. Match scoring is explicitly deferred to v0.2.

---

## 3. Directory Layout

```
plugins/hr_recruit/
├── plugin.json                  # manifest: capabilities/menu/settings/permissions/hooks
├── __init__.py                  # HRRecruitPlugin(BasePlugin)
├── models.py                    # connection factory (pool + search_path + return)
├── crypto.py                    # envelope encryption: DEK/KEK & key destruction
├── llm.py                       # UnifiedLLM wrapper (tier + prompt version)
├── jd.py                        # JD generation 4-stage + human sign-off
├── bias.py                      # dual-channel bias review + impact ratio
├── talent_search.py              # talent pool search (vector / keyword fallback)
├── scheduling.py                # scheduling & stage notifications
├── retention.py                  # retention scan & purge
├── dag_nodes.py                 # DAG node handler set
├── hr_skill.py                   # self-contained skill (for chat/workflow)
├── routes.py                    # Blueprint /admin/hr-recruit/*
├── parsers/
│   └── resume_parser.py         # pure-Python resume parsing (no external binaries)
├── tools/
│   └── mcp_server.py            # MCP tool server (stdio JSON-RPC)
├── migrations/
│   └── v1.0.0_init.sql          # hr_recruit schema DDL
├── templates/
│   ├── hr_recruit.html          # inline partial (first line JS, no script tags)
│   └── hr_recruit_iframe.html   # iframe full page (with theme sync)
├── i18n/
│   ├── en.yml                   # identity mapping (required)
│   └── zh-CN.yml                # Chinese mapping (required)
├── agents/
│   └── hr_recruiter_prompt.md
├── agent_matrix_additions/      # legacy reference (archived since v1.0.0)
├── tests/                       # unit & regression tests
├── docs/                        # compliance, acceptance, deployment docs
├── SKILL.md / USAGE.md / README.md / CHANGELOG.md
└── requirements.txt / setup.sh
```

---

## 4. Installation & Configuration

### 4.1 Deployment

```bash
# 1) Copy plugin directory
cp -r hr_recruit_plugin <repo>/plugins/hr_recruit

# 2) Install dependencies & self-check
bash <repo>/plugins/hr_recruit/setup.sh

# 3) Restart admin service
#    (capabilities auto-aggregate to core role office via
#     plugin.json agent_role=office at startup, no manual YAML copying)
```

### 4.2 Required configuration

| Setting | Description |
|---------|-------------|
| `hr_master_key` (KEK, **required**) | Master key wrapping each candidate's DEK. **Plugin refuses to enable if unset** |
| `llm_tier` | `high`/`standard`/`cheap`/`local`; `local` keeps resumes off the internal network |
| `resume_raw_days` | Raw resume retention, default 180 days |
| `candidate_pii_days` | Identity field retention, default 365 days |
| `max_upload_mb` | Resume upload size limit, default 10 MB |
| `purge_mode` | `crypto_shred` (default: destroy key + mask) / `hard_delete` (also deletes application chain & master record) |
| `notify_before_days` | Days to notify before purge, default 7 |
| `bias_scan_rate_limit` | Bias review rate limit (req/min), default 30 |
| `resume_upload_rate_limit` | Resume upload rate limit (req/min), default 10 |
| `work_timezone` | Working hours timezone, default `Asia/Shanghai` |

> **Decision log is permanent and append-only** (R-8): `hr_decision_log` stores only masked `ref_hash`, never deleted by age, automatically meeting AI Act ≥180-day log retention.
>
> **KEK loss = all candidate PII permanently unreadable.** Complete backup and handoff before enabling.

---

## 5. Data Model (`hr_recruit` schema)

| Table | Purpose |
|-------|---------|
| `hr_job` / `hr_job_description` | Positions & JD version chain (with reviewer questions, human signer) |
| `hr_candidate` | Candidate master record (PII fields DEK-encrypted) |
| `hr_candidate_dek` | Data keys; **deleting this row = all copies unreadable** |
| `hr_resume_raw` | Raw resume & extracted text (encrypted, shortest retention) |
| `hr_candidate_profile` | Structured profile (non-PII, plaintext for search/stats) |
| `hr_talent_embedding` | Talent pool vectors (dimension corrected at runtime, never hardcoded) |
| `hr_application` | Application/process instance (`source`: resume_upload/talent_search/manual; one active application per candidate per job) |
| **`hr_decision_log`** | **Compliance core: decision log, PG RULE forbids UPDATE/DELETE** |
| `hr_bias_metric` | Impact ratio statistics (NYC LL144) |
| `hr_purge_receipt` | Purge receipt (audit evidence) |

---

## 6. Retention Purge (crypto-shredding)

Four layers of residue: the first three (main DB / files / vectors) can be precisely located and deleted; **the fourth layer (vault backups & snapshots) cannot be individually located** — this is where most HR systems fail.

This plugin's approach: **don't chase copies, make them all undecryptable**:

```
Write: each candidate gets an independent DEK → PII encrypted with DEK → DEK wrapped by KEK into hr_candidate_dek
Purge: delete DEK row → ciphertext in main DB / vector copies / vault backups / incremental snapshots becomes permanently undecryptable
```

Execution order iron rule: **destroy the key first, then delete data** (reversing this leaves plaintext residue on failure).

**Key derivation & ciphertext version (v2, B12)**

| Item | v2 (new data, default) | v1 (legacy, read-only) |
|------|------------------------|------------------------|
| KEK | `PBKDF2-HMAC-SHA256(master, salt, 200000)` → 32B | Master key zero-padded/truncated to 32B |
| Ciphertext format | `0x02 ‖ nonce ‖ ciphertext` | `nonce ‖ ciphertext` |
| AAD binding | Candidate ID + purpose (field-level) / candidate ID (DEK-level) | None |
| `kek_id` | `hr_recruit_kek_pbkdf2_v2` | `hr_recruit_default_kek` |

Upgrade **does not migrate data or re-encrypt**: decrypt/unpack tries v2 first, falls back to v1 on failure; legacy ciphertext remains readable (Option A: versioned fallback). PBKDF2 results are cached per master key material, one derivation cost per worker.

> **Relationship with vault**: purge does not rewrite historical backups. PII of purged candidates still exists as ciphertext in historical backups, but the DEK is destroyed so it's permanently unreadable. Restoring from a vault backup will **not** resurrect purged PII.

---

## 7. API

All endpoints under `/admin/hr-recruit/api/*`, unified response `{"success": bool, ...}`.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stats` | Dashboard statistics |
| GET/POST | `/api/jobs` | Job list / create |
| POST | `/api/jobs/<id>/jd` | Generate JD + bias review |
| GET | `/api/jobs/<id>/jd/versions` | JD version chain |
| POST | `/api/jobs/<id>/jd/<v>/signoff` | **Human sign-off** (override requires a reason, logged separately as `jd_publish_override`) |
| POST | `/api/bias/scan` | Bias review (rate-limited; exemptions judged by text semantics, no `job_category` parameter). Dual-channel gate: returns `channels` (rule/semantic status) and `coverage` (dual / rule_only); when rule channel has zero hits but semantic channel is not run/failed/uncovered, **must not return "pass"** (fail-closed) |
| POST | `/api/resume/parse` | Resume upload & parse (rate-limited; `max_upload_mb` size & page limits; requires `consent`) |
| POST | `/api/talent/search` | Talent pool search |
| POST | `/api/schedule/propose` | Schedule proposal (frontend form: interviewer busy slots / holidays / duration / window, pure computation, no DB write) |
| GET | `/api/applications` | Pipeline metadata (no PII) |
| POST | `/api/applications` | Create application (validates job open / candidate exists / same-job dedup, writes evidence triad start) |
| POST | `/api/applications/<id>/stage` | Advance stage (no skips/rollbacks; terminal hired/rejected auto-closed and logged) |
| GET | `/api/retention/stats` · `/scan` · `/receipts` | Retention stats / scan / receipts |
| POST | `/api/retention/purge` | Manual purge. Single candidate: must type `ref_hash` (server compares verbatim, rejects on mismatch); batch: requires `confirm: true` and ≤ `purge_batch_max` (default 50) per call |
| GET | `/api/candidates` · `/api/candidates/<id>` · `/api/candidates/<id>/resume` | Candidate list / detail / resume (detail decryption logs a view each time) |
| GET | `/api/decisions` | Decision log (masked) |
| GET | `/api/metrics` | Hiring metrics: stage funnel (`hr_application.stage`), channel distribution (`source`), time-to-close (avg `closed_at - applied_at` days). All from existing columns, no new tables, no PII |
| GET | `/api/notice` | Candidate notice documents: current version + version list + body (bilingual). Frontend must not hardcode versions; `/api/resume/parse` validates `notice_version` against the list |

## 8. MCP Tools

`plugin.json` registers `mcp_servers[0].name = "hr"`, platform exposes as `mcp__hr_recruit__hr__*`:

| Tool | Capability | Description |
|------|-----------|-------------|
| `hr_job_list` | Jobs | Open positions list |
| `hr_jd_versions` | JD | JD version chain for a position (version/model/signoff status/summary), read-only |
| `hr_bias_scan` | Bias review | Discriminatory phrasing rule scan (local fast path, no LLM) |
| `hr_talent_search` | Talent search | Semantic search (masked candidate set) |
| `hr_schedule_propose` | Scheduling | Cross-calendar free-slot proposal (pure computation, no DB write) |
| `hr_retention_stats` | Retention | Statistics (due / upcoming / purged) |
| `hr_decision_log` | Audit trail | Recent decision log (masked, only `candidate_ref`) |

**Exposure boundary**: MCP layer only exposes fast-path and read-only/stats tools. Kernel `McpClient` has a default 15s timeout with no per-plugin config, so **long LLM calls (JD generation / resume parsing) and irreversible actions (manual purge) do not go through MCP** — they go via HTTP API or `hr_skill` (the latter runs synchronously inside the host Agent under policy constraints).

**Compliance iron rule: MCP tools never return candidate PII plaintext.**

## 9. DAG Nodes

`hr.jd_draft` · `hr.bias_review` · `hr.resume_parse` · `hr.talent_search` · `hr.schedule` · `hr.purge`

> ⚠ Kernel `approval` / `sub_workflow` / `script` nodes are currently stubs. v0.1 uses "human sign-off API + decision log" in place of hire approval; will switch when the kernel fills them in.

---

## 10. Related Docs

- `docs/00-index.md` — doc index & deployment checklist
- `docs/compliance.md` — compliance matrix & self-check
- `docs/acceptance.md` — functional & compliance acceptance checklist
- `USAGE.md` — usage guide
- `SKILL.md` — skill description
- `tests/REGRESSION_TESTING.md` — regression testing notes
