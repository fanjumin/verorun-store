# NeuralFlow (`neural_flow`)

> Platform-level AI dataflow observability base. Records NeuralFlow spans,
> loads Domain Profiles, and supports cross-domain replay for the NeuralHub
> dashboard.

- **Identifier:** `neural_flow`
- **Version:** 1.1.0
- **Category:** ai_agent
- **Min app version:** 0.10.0
- **Agent role:** `ops`
- **Python deps:** none (optional `pyyaml` for external Domain Profile scan)

---

## Core Features

### 1. Span collection (`collectors.py`, 7.8 KB)
- Incremental cursor poll on platform tables (agent_token_logs etc.).
- `collector_interval_seconds=5`.
- Spans in `nf_flow_spans`.

### 2. Span retention
- `span_retention_days=30` — daily prune.

### 3. Domain Profile (`profile_registry.py`, 13.3 KB)
- Loads external Domain Profiles (requires pyyaml).
- Without pyyaml, only built-in profiles available.

### 4. Cross-domain replay
- `sdk.py` (6 KB) — SDK.
- `sse_stream.py` (7.5 KB) — SSE push.

### 5. Hooks listened
- `agent.task.completed`, `plugin.installed`, `plugin.enabled`,
  `plugin.disabled` — auto-update profile on plugin install / toggle.

### 6. Execution-level observability (v1.1.0)
- **Execution join (`join.py`)** — `join_execution()` merges traces from
  `public.agent_tasks` / `agent_token_logs` / `nf_flow_spans` by
  `trace_id` (= `task_id` priority, falls back to `module`). Read-only over
  upstream tables; results are recomputable derived data.
  `joined_by` marks the join quality: `task_id` (full join) vs
  `module_fallback` (task-level degraded view, no spans).
- **Per-trace execution view** — `obs_execution_view` holds outcome
  (7-state status map), duration, total tokens, confidence and the derived
  tree (`parent_task_id` / `master_task_id`). `duration_ms` uses
  `agent_tasks.started_at` / `completed_at` and falls back to span `ts`
  spread; it never uses the nonexistent `agent_token_logs.elapsed_ms`.
- **Stage attribution** — `obs_attribution` aggregates spans per `stage`
  with dual shares: `share_of_total_tokens` vs `share_of_duration`
  (mismatch is the insight). Anchor spans (`event='anchor'`) are excluded.
- Endpoints: `GET /api/executions`, `GET /api/executions/<trace_id>`,
  `GET /api/attribution/<trace_id>`.
- Migration: `migrations/v1.1.0_observatory.sql` (idempotent).

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `neural_flow`, **3 tables** |
| Blueprint | `/api/neural-flow` (`routes.py`, 7 KB) |
| Collectors | `collectors.py` (7.8 KB) |
| Profile | `profile_registry.py` (13.3 KB) |
| SSE | `sse_stream.py` (7.5 KB) |
| SDK | `sdk.py` (6 KB) |
| Python deps | none (optional pyyaml) |
| Permissions | routes, scheduler, events |
| Capabilities | neural_flow.base, neural_flow.profile, neural_flow.obs_execution, neural_flow.obs_attribution |
| Hooks listened | agent.task.completed, plugin.installed/enabled/disabled |

### 3 tables

| Table | Purpose |
|---|---|
| `nf_flow_spans` | Dataflow spans (30-day archive) |
| `obs_execution_view` | Per-trace joined execution view (derived, v1.1.0) |
| `obs_attribution` | Per-stage attribution with dual shares (derived, v1.1.0) |

### Config (2 items)

| Key | Default | Purpose |
|---|---|---|
| `collector_interval_seconds` | 5 | Collection poll interval |
| `span_retention_days` | 30 | Span retention days |

---

## Five-minute quick start

1. Enable — auto-creates table.
2. Install pyyaml (optional, for external Domain Profile scan).
3. Wait for collector to poll platform tables.
4. NeuralHub dashboard shows spans.
5. Plugin install / toggle auto-updates profile.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Platform engineers / SRE | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
