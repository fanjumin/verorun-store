# CogEvolution (`memory_engine`)

> Hierarchical agent memory. Vector retrieval, Reflexion-based self-
> evolution, sedimentation of high-value memories into the shared KB,
> injection A/B testing, and memory graph visualization. Conflicts with
> `cogevolution_substrate`.

- **Identifier:** `memory_engine`
- **Version:** 1.8.0
- **Category:** system
- **Menu group:** System → CogEvolution
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, research, minipro
- **Agent role:** `athena`
- **Conflicts:** `cogevolution_substrate`
- **Python deps:** none

---

## Core Features

### 1. Auto extraction (`services/extractor.py`, 14.1 KB)
- Listens to `agent.task.completed`.
- `enable_auto_extract=true`, `daily_extract_budget=200`.
- `max_memories_per_owner=500`, `retention_days=365`.
- `max_memory_block_len=1200`.

### 2. Vector retrieval
- `top_k=5` (1–20).
- Hooks `before_prompt_resolve` — injects relevant memories at prompt time.

### 3. Reflexion (`services/reflexion.py`, 6.9 KB)
- `enable_reflexion=true`, `reflexion_failure_only=true`.
- `reflexion_min_confidence=0.4`.

### 4. Sedimentation (`services/sedimentation.py`, 14.3 KB)
- Promotes high-value memories to shared KB.
- `sedimentation_fact_min_confidence=0.9`.
- `sedimentation_lesson_min_rating=4` (1–5).
- `sedimentation_min_quality_score=0.7`.
- `sedimentation_daily_budget=50`.
- Approval: `POST /sedimentation/<id>/approve` / `reject`.

### 5. Prompt evolution (`services/prompt_evolution.py`, 8.7 KB)
- `prompt_evolution_enabled=false` (default off).

### 6. Forgetting curve (`services/forgetting.py`, 7.2 KB)
- Decays memory weights along a forgetting curve.

### 7. Memory graph
- `GET /graph` — visualize memory relations
  (`templates/memory_admin_graph.html`, 10 KB).

### 8. A/B testing
- `abtest_enabled=false`, `abtest_control_pct=50`.
- 50/50 deterministic hash split to measure injection benefit.
- `GET /abtest/report`.

### 9. Eval cases
- `GET/POST /eval/cases`, `POST /eval/run`, `GET /eval/runs`.

### 10. User opt-in
- `memory_opt_in_default=true` (default on for new users).
- `GET/PUT /my/optin`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `memory_engine` (schema_version metadata) |
| Blueprints | `/admin/memory` (`routes.py`, 24.3 KB) + `/api/my-memory` |
| HTTP endpoints | **21** |
| Services | sedimentation.py (14 KB), extractor.py (14 KB), prompt_evolution.py (8.7 KB), forgetting.py (7.2 KB), reflexion.py (6.9 KB) |
| Admin UI | memory_admin.html (8.7 KB) + memory_admin_graph.html (10 KB) |
| Python deps | none |
| Permissions | api:read/write, user:profile, admin:access |
| Capabilities | memory.extract, memory.reflexion, memory.sedimentation |
| Hook provided | before_prompt_resolve |
| Hooks listened | agent.task.completed |
| Dashboard | memories_total, reflexions_total, injections_today, evolution_suggestions |

### Key config (18 items, common subset)

| Key | Default | Purpose |
|---|---|---|
| `top_k` | 5 | Recall (1–20) |
| `enable_auto_extract` | true | Auto-extract |
| `daily_extract_budget` | 200 | Daily extract budget |
| `max_memories_per_owner` | 500 | Per-owner cap |
| `retention_days` | 365 | Retention days |
| `enable_reflexion` | true | Reflexion |
| `reflexion_failure_only` | true | Failures only |
| `reflexion_min_confidence` | 0.4 | Min confidence |
| `memory_opt_in_default` | true | New-user opt-in default |
| `allow_global_memory` | false | Cross-user global memory |
| `prompt_evolution_enabled` | false | Prompt evolution suggestions |
| `enable_sedimentation` | true | Knowledge sedimentation |
| `sedimentation_fact_min_confidence` | 0.9 | Fact threshold |
| `sedimentation_lesson_min_rating` | 4 | Lesson rating threshold |
| `sedimentation_daily_budget` | 50 | Daily sediment budget |
| `abtest_enabled` | false | Injection A/B |
| `abtest_control_pct` | 50 | Control group percent |

---

## Five-minute quick start

1. Enable — auto-creates tables (note: conflicts with cogevolution_substrate).
2. Open **System → CogEvolution**.
3. Watch memories_total / injections_today.
4. Sedimentation queue → approve high-value memories into shared KB.
5. Optionally enable A/B to measure injection benefit.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | AI engineers / ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
