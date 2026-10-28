# Evolution Core (`cogevolution_substrate`)

> Industrial data-flywheel substrate for AI agents. Curates memories from
> task outcomes, runs reflexion on failures, evolves agent prompts with
> rule- or model-based strategies, supports canary rollout, detects
> distribution drift, and labels unlabelled records. Conflicts with
> `memory_engine`.

- **Identifier:** `cogevolution_substrate`
- **Version:** 1.5.0
- **Category:** ai_agent
- **Menu group:** AI & Agents → Evolution Core
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, minipro, edge
- **Agent role:** `athena`
- **Conflicts:** `memory_engine`
- **Python deps:** none

---

## Core Features

### 1. Memory curation
- `services/curation_store.py` (4.8 KB) — curation event store.
- `services/extractor.py` (5 KB) — listens to `agent.task.completed`,
  extracts within `daily_extract_budget=200`.
- `services/retriever.py` (3.9 KB) — top-k=5 recall.
- 3 curation prompts: `agents/curation_curator_prompt.md`,
  `agents/evolution_curator_prompt.md`, `agents/label_curator_prompt.md`.

### 2. Reflexion
- `services/reflexion.py` (2.3 KB) — post-mortem on failures.
- `reflexion_failure_only=true`: only failed tasks.
- `reflexion_min_confidence=0.4`: confidence floor.

### 3. Prompt evolution
- `services/prompt_evolution.py` (5.3 KB) — two strategies:
  - `rule` (default): rule-driven.
  - `model`: LLM-driven.
- `services/prompt_injector.py` (3.5 KB) — hooks
  `before_prompt_resolve`, injects evolved fragments at prompt time.
- `evolvable_agents`: agent slug whitelist that may receive patches.

### 4. Canary rollout
- `services/canary.py` (5.2 KB) — `evolution_rollout_mode=append`
  (all users) or `canary`.
- `canary_pct=10`: 10% of users get patches first (deterministic hash).
- `canary_min_samples=8`: minimum before/after samples before regression
  check.
- `canary_device_pct=50`: 50% of devices for IIoT.
- Auto-rollback on regression.

### 5. Shadow mode
- `services/shadow.py` (15 KB) — offline strategy comparison, no
  production impact.

### 6. Drift detection
- `services/drift.py` (7.1 KB) — z-score based.
- `drift_z_threshold=3.0`: |z| > 3 treated as drift.
- `drift_min_samples=30`: per-signal baseline samples needed.
- Baselines in `device_signal_baseline`.

### 7. Labelling dataset
- `services/labeling.py` (7.2 KB) — sends unlabelled records to label
  curator.
- `label_scan_days=7`: look-back window.
- `services/dataset.py` (2.8 KB) + `services/training.py` (0.6 KB) —
  dataset export and training prep.
- `dataset_exports` table.

### 8. Ingest bindings (3)
- `bindings/agent_domain.py` (0.7 KB) — agent domain.
- `bindings/knowledge.py` (1.7 KB) — knowledge blocks.
- `bindings/sim_telemetry.py` (1.8 KB) — IIoT telemetry.
- Config: `enable_knowledge_ingest=true` (10 min loop),
  `enable_iiot_ingest=false`, `iiot_ingest_mode=all/drift`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `cogevolution_substrate`, **7 tables** |
| Blueprint | `/admin/cogevolution` (`routes.py`, 10.2 KB, serves admin SPA) |
| Admin UI | `templates/cogevolution_admin.html` (8 KB) |
| Internal services | 13 .py under `services/` |
| Hook provided | `before_prompt_resolve` |
| Hooks listened | `agent.task.completed`, `cogevolution.improvement.outcome` |
| Permissions | `api:read`, `api:write`, `user:profile`, `admin:access`, `routes`, `events`, `scheduler` |
| Capabilities | `memory.curate`, `memory.extract`, `memory.iiot` |
| Dashboard | 6 stats: memories_total, reflexions_total, injections_today, evolution_suggestions, drift_hits_total, label_suggestions_pending |

### 7 tables

| Table | Purpose |
|---|---|
| `curation_events` | Curation events (memory extraction) |
| `improvement_events` | Improvement events (prompt patches) |
| `improvement_feedback` | Improvement feedback |
| `improvement_audit` | Improvement audit |
| `device_signal_baseline` | Device signal baseline (drift detection) |
| `label_suggestions` | Pending label suggestions |
| `dataset_exports` | Dataset export records |

### Key config (26 items, common subset)

| Key | Default | Purpose |
|---|---|---|
| `enable_auto_extract` | true | Auto-extract memories |
| `daily_extract_budget` | 200 | Daily extract budget |
| `max_memories_per_owner` | 500 | Per-owner memory cap |
| `enable_reflexion` | true | Enable reflexion |
| `reflexion_failure_only` | true | Only on failed tasks |
| `top_k` | 5 | Recall count (1–20) |
| `reflexion_min_confidence` | 0.4 | Min reflexion confidence |
| `evolution_strategy` | rule | rule / model |
| `evolvable_agents` | [] | Whitelist of agent slugs |
| `evolution_rollout_mode` | append | append / canary |
| `canary_pct` | 10 | Canary percent |
| `canary_min_samples` | 8 | Min samples before regression check |
| `drift_z_threshold` | 3.0 | Drift z threshold |
| `drift_min_samples` | 30 | Min baseline samples |
| `enable_knowledge_ingest` | true | Knowledge ingest |
| `knowledge_ingest_interval_min` | 10 | Knowledge ingest interval |
| `enable_iiot_ingest` | false | IIoT telemetry ingest |
| `iiot_ingest_mode` | all | all / drift |
| `label_scan_days` | 7 | Label look-back window |

---

## Five-minute quick start

1. Enable — runs migrations v0.1.0 → v0.5.0 (5 SQL files).
2. Open **AI & Agents → Evolution Core**.
3. Set `evolvable_agents` (agent slugs allowed to receive patches).
4. Review **Suggestions** in the dashboard; approve / reject.
5. Switch `evolution_rollout_mode=canary` for safer rollout.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / integrator | `README.md` |
| Technical whitepaper | ML / agent engineers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops / ML | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
