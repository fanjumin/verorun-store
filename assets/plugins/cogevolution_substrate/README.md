# Evolution Core · 通用进化内核

`cogevolution_substrate` — v1.0.1 · VeroRun Official Plugin · `ai_agent` · free

> **English:** An industrial data-flywheel evolution substrate for AI agents. It curates
> task outcomes into memory, reflects on failures, and produces human-gated improvement
> proposals that can be applied to live agent prompts.
>
> **中文：** 面向工业 / 机器学习 / 数据飞轮场景的智能体受控进化基座。它把智能体任务产出
> 策展为记忆、对失败进行反思，并生成经人工闸门审批后才能落地的进化提案。

---

## Table of Contents · 目录

1. [Overview · 概述](#overview--概述)
2. [Features · 核心特性](#features--核心特性)
3. [How It Works · 工作原理](#how-it-works--工作原理)
4. [Architecture · 架构与目录](#architecture--架构与目录)
5. [Database Schema · 数据库结构](#database-schema--数据库结构)
6. [Installation & Enablement · 安装与启用](#installation--enablement--安装与启用)
7. [Configuration · 配置项](#configuration--配置项)
8. [Admin Dashboard · 管理界面](#admin-dashboard--管理界面)
9. [API Reference · 管理 API](#api-reference--管理-api)
10. [Events & Hooks · 事件与钩子](#events--hooks--事件与钩子)
11. [Privacy & Security · 隐私与安全](#privacy--security--隐私与安全)
12. [Relation to memory_engine · 与 memory_engine 的关系](#relation-to-memory_engine--与-memory_engine-的关系)
13. [Known Limitations · 已知限制](#known-limitations--已知限制)
14. [Development & Tests · 开发与测试](#development--tests--开发与测试)

---

## Overview · 概述

**English**

`cogevolution_substrate` is a self-contained evolution substrate that lives in its own
PostgreSQL schema (`cogevolution_substrate`) and never touches VeroRun core tables. It
implements a closed loop:

```
agent task completed → extract → curate → store memory
        ↑                                 ↓
      inject                          reflexion
        ↑                                 ↓
   (on next prompt)              shadow proposal → human approve → apply → audit
```

Everything it records is **owner-scoped** (`owner_type` / `owner_id`) and gated by user
opt-in and an explicit `cognitive_engine` selection, so it coexists safely with the
`memory_engine` plugin (see [Relation](#relation-to-memory_engine--与-memory_engine-的关系)).

**中文**

`cogevolution_substrate` 是一个独立进化基座，数据保存在独立的 PostgreSQL schema
`cogevolution_substrate` 中，不触碰 VeroRun 核心表。它实现了完整的进化闭环：

```
智能体任务完成 → 提取 → 策展 → 存储记忆
        ↑                                 ↓
     注入记忆                       失败反思
        ↑                                 ↓
   （下次提示词）            影子提案 → 人工审批 → 应用 → 审计
```

所有记录都按属主隔离（`owner_type` / `owner_id`），并受用户主动选择与
`cognitive_engine` 闸门控制，可与 `memory_engine` 插件安全共存。

---

## Features · 核心特性

| Feature · 特性 | English | 中文 |
|---|---|---|
| Memory Curation · 记忆策展 | Curates task results into deduplicated, keyword-tagged memory records. | 把任务结果策展为去重、带关键词标签的记忆记录。 |
| Reflexion · 失败反思 | Rule-based lesson extraction on failed / low-confidence tasks. | 对失败/低置信度任务按规则提取经验教训。 |
| Prompt Injection · 提示词注入 | Injects the most relevant memories into the next agent prompt. | 把最相关的记忆注入下一次智能体提示词。 |
| Self-Evolution · 自进化 | Shadow improvement proposals with a human approval gate; real persistence on apply. | 影子进化提案 + 人工审批闸门，批准后真实落盘。 |
| Audit Trail · 审计追踪 | Every proposal state change is recorded in `improvement_audit`. | 每个提案状态变更都写入 `improvement_audit`。 |
| pgvector Optional · pgvector 可选 | Vector retrieval when the `vector` extension exists, keyword fallback otherwise. | 存在 `vector` 扩展时用向量检索，否则自动回退关键词。 |
| Admin UI · 管理界面 | Dashboard, proposal approve/reject/apply, and curation browsing. | 仪表盘、提案审批/应用、记忆浏览。 |

---

## How It Works · 工作原理

### 1. Extraction · 提取

The plugin listens on the framework event `agent.task.completed`. When a task completes:

- `AgentDomainBinding.to_curation()` maps the result dict into a curation record
  (`content` = `response` or `error`, `source_id` = `task_id`).
- `MemoryExtractor.on_event()` filters the record:
  - skips content shorter than 4 chars or matching PII patterns,
  - skips greetings (`hello`, `hi`, `thanks`, ...),
  - enforces the daily per-owner budget (`daily_extract_budget`, default 200),
  - only runs when the user has selected `cognitive_engine = cogevolution_substrate`
    and opted into memory (`memory_opt_in`).
- `CurationStore.ingest()` writes a deduplicated record (SHA-256 `content_hash` scoped
  by owner + domain) and enforces the per-owner cap
  (`max_memories_per_owner`, default 500; lowest-importance records are archived).

### 2. Injection · 注入

`PromptInjector` registers a `before_prompt_resolve` hook filter (priority 10). When
resolving a prompt it:

1. verifies the user selected this engine and opted in (fails closed on any profile error),
2. calls `MemoryRetriever.build_injection_block()` to fetch the top-`k` memories
   (`top_k`, default 5),
3. appends the block to the prompt inside an `=== Evolution Core (auto) ===` fence.

The retriever tries vector similarity first (`embedding <=> ?::vector`) when the
`vector` extension is available, and falls back to `content ILIKE` keyword search.
Every injected row bumps `hit_count` and `last_hit_at` (used by "Injections Today" stats).

### 3. Reflexion · 反思

`ReflexionService` also listens on `agent.task.completed` but is independent of the
extractor (reentrancy-guarded via `threading.local`). It only triggers when:

- `enable_reflexion` is true,
- the user chose this engine and opted in,
- the task **failed**, or (when `reflexion_failure_only` is false) confidence is below
  `reflexion_min_confidence` (default 0.4).

It stores a `lesson`-type record: `任务失败/低置信度：<query>。改进线索：<detail>`.

### 4. Self-Evolution · 自进化

- A daily cron job (`03:20`) calls `PromptEvolutionService.propose()` for the first slug
  in `evolvable_agents`. It is skipped when a proposal already exists for that target
  today (dedup), or when one was already applied.
- `ShadowService.propose()` writes an `improvement_events` row. Proposals with
  confidence < 0.6 are auto-`rejected` (shadow-only).
- State machine: `proposed → accepted → applied` (or `rejected`).
- `apply()` (with `confirmed=true`) **really persists** the evolution: it appends
  `\n[EVOLUTION] <summary>` to `agent_matrix.system_prompt` for the target agent slug.
- `rollback()` reverts applied proposals for a target back to `rejected`.
- Every transition is written to `improvement_audit` and emitted as a
  `cogevolution.improvement.*` event.

---

## Architecture · 架构与目录

```
plugins/cogevolution_substrate/
├── __init__.py                     # Plugin entry point: lifecycle, events, cron
├── plugin.json                     # Manifest (menu, config, permissions, agents)
├── models.py                       # DB access, migrations, pgvector detection
├── routes.py                       # Admin REST API (JWT admin-gated)
├── agents/
│   └── curation_curator_prompt.md  # Sub-agent prompt for Curation Curator
├── bindings/
│   ├── __init__.py
│   ├── agent_domain.py             # Maps agent task results → curation records
│   └── sim_telemetry.py            # (legacy stub, not active)
├── services/
│   ├── curation_store.py           # Dedup/embedding/limit-aware persistence
│   ├── extractor.py                # PII + budget + opt-in extraction boundary
│   ├── retriever.py                # Vector-with-keyword-fallback retrieval
│   ├── prompt_injector.py          # before_prompt_resolve hook filter
│   ├── reflexion.py                # Rule-based lesson extraction
│   ├── prompt_evolution.py         # Shadow proposal generation
│   ├── shadow.py                   # Proposal state machine + apply/rollback
│   ├── metrics.py                  # outcome feedback aggregation
│   ├── labeling.py                 # Human labeling service
│   └── training.py                 # (legacy stub, not active)
├── migrations/                     # Lexically ordered, idempotent SQL
├── templates/
│   └── cogevolution_admin.html     # Admin SPA partial (window.l_cogevolution_admin)
├── i18n/
│   ├── en.yml
│   └── zh-CN.yml
├── static/
└── tests/                          # test_pii.py, test_injection.py
```

---

## Database Schema · 数据库结构

All tables live in the dedicated `cogevolution_substrate` schema. Migrations are
applied lexically via `schema_version` and are idempotent.

### `curation_events` · 记忆事件

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `owner_type` / `owner_id` | varchar | owner scope (default `user` / `''`) |
| `domain_id` | varchar | e.g. `agent` |
| `record_type` | varchar | `fact` (memory) or `lesson` (reflexion) |
| `content` | text | max 10000 chars, checked |
| `keywords` | text[] | auto-extracted (up to 12) |
| `embedding` | vector(1536) | **optional** — only when `vector` extension exists |
| `source` / `source_id` | varchar | e.g. `auto` / `task_id` |
| `content_hash` | varchar UNIQUE | SHA-256, owner+domain scoped dedup |
| `confidence` / `importance` / `quality_score` | real | 0..1 |
| `label` / `labels_count` | jsonb / int | human labels |
| `status` | varchar | `active` / `archived` |
| `hit_count` / `last_hit_at` | int / timestamptz | injection tracking |
| `created_at` / `updated_at` | timestamptz | |

### `improvement_events` · 进化提案

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `owner_type` / `owner_id` | varchar | owner scope |
| `domain_id` / `scope` | varchar | e.g. `agent` / `strategy` |
| `target_ref` | varchar | target agent slug for apply |
| `summary` | text | the evolution patch text |
| `rationale` | jsonb | |
| `confidence` | real | < 0.6 auto-rejected |
| `shadow` | bool | true until applied |
| `status` | varchar | `proposed` → `accepted` → `applied` / `rejected` |
| `applied_at` | timestamptz | |
| `created_at` | timestamptz | |

### `improvement_feedback` · 反馈指标

Outcome metrics per proposal (`sample_count`, `success_count`, `success_rate`,
`avg_rating`, `total_tokens`).

### `improvement_audit` · 审计

Action log: `proposed` / `accepted` / `applied` / `rejected` / `rollback` with actor and
JSON detail.

### `schema_version` · 迁移版本

Migration bookkeeping (`version` = filename, `applied_at`).

**pgvector degradation · pgvector 降级：** On install, `migrate_all()` checks
`pg_available_extensions`; if the `vector` extension is available it is created and the
`embedding` column is added, enabling vector search. Otherwise the column is omitted and
retrieval automatically uses keyword matching — no code change required.

---

## Installation & Enablement · 安装与启用

- **Requirements:** VeroRun ≥ 0.59.3, PostgreSQL (psycopg2). pgvector is optional.
- **Install:** The plugin ships with VeroRun and is installed/enabled from the Admin →
  Plugins store, or via `plugin_manager` directly.
- On enable, `models.migrate_all()` creates the schema and runs migrations.
- On uninstall, `drop_schema()` removes only plugin-owned tables (`DROP SCHEMA ... CASCADE`).

```bash
# Example (inside a VeroRun Python shell)
from plugin_manager import get_manager
m = get_manager()
m.install_plugin('cogevolution_substrate')
m.enable_plugin('cogevolution_substrate')
```

---

## Configuration · 配置项

Managed in Admin → Plugins → Evolution Core → Settings (also available as
`config` in `plugin.json`).

| Key | Default | Description |
|---|---|---|
| `enable_auto_extract` | `true` | Auto-curate task results into memory. |
| `daily_extract_budget` | `200` | Max extractions per owner per day. |
| `max_memories_per_owner` | `500` | Cap on active memories per owner (oldest/least-important archived). |
| `memory_opt_in_default` | `false` | Default opt-in when the user profile has no value. |
| `enable_reflexion` | `true` | Enable rule-based reflexion. |
| `reflexion_failure_only` | `true` | Reflect on failed tasks only. |
| `reflexion_min_confidence` | `0.4` | Confidence threshold when `reflexion_failure_only=false`. |
| `max_memory_block_len` | `800` | Max chars of the injected memory block. |
| `coexistence_mode` | `remind` | `remind` = respect per-user engine choice; `force_default` = use `default_engine` unless user chose otherwise. |
| `default_engine` | `memory_engine` | Fallback engine for `force_default`. |
| `top_k` | `5` | Memories injected per prompt. |
| `evolution_strategy` | `rule` | Proposal strategy (`rule` only). |
| `evolvable_agents` | `[]` | **Agent slugs allowed to receive prompt-evolution patches.** Empty = no daily proposals. |

> **Security note:** a proposal can only ever modify the `system_prompt` of an agent slug
> explicitly listed in `evolvable_agents`.

---

## Admin Dashboard · 管理界面

After enabling, the menu **AI & Agents → Evolution Core** opens the admin SPA
partial `templates/cogevolution_admin.html` (`window.l_cogevolution_admin`), showing:

- **Stats:** Total Memories / Reflexions / Injections Today / Suggestions.
- **Evolution Proposals:** summary, target agent, scope, status, confidence, and
  **Approve / Reject / Apply** actions.
- **Curation Records:** type, content, owner, domain, quality, status.

---

## API Reference · 管理 API

All endpoints require a Bearer token (or `sso_token` / `tm_token` cookie) whose JWT
carries `is_admin: true`; otherwise `401`.

| Method | Path | Description |
|---|---|---|
| GET | `/admin/cogevolution` | Dashboard stats. |
| GET | `/admin/cogevolution/curation?owner_type=user&owner_id=` | List curation records. |
| POST | `/admin/cogevolution/curation/<event_id>/label` | Human-label a record. |
| GET | `/admin/cogevolution/improvements` | List improvement proposals. |
| POST | `/admin/cogevolution/improvements/<id>/approve` | Accept a proposal (`{confirmed:true}`). |
| POST | `/admin/cogevolution/improvements/<id>/reject` | Reject a proposal. |
| POST | `/admin/cogevolution/improvements/<id>/apply` | Apply the patch to the target agent prompt. |
| POST | `/admin/cogevolution/improvements/<id>/outcome` | Report success/rating/tokens. |
| POST | `/admin/cogevolution/improvements/rollback` | Revert applied proposals for a `target_ref`. |

---

## Events & Hooks · 事件与钩子

**Listens · 监听**

- `agent.task.completed` — extraction + reflexion.
- `cogevolution.improvement.outcome` — feedback aggregation.

**Provides · 提供**

- `before_prompt_resolve` (hook filter, priority 10) — memory injection.

**Emits · 发出**

- `cogevolution.improvement.proposed` / `.accepted` / `.applied` / `.rejected`

---

## Privacy & Security · 隐私与安全

- **PII blocking:** passwords, API keys, secrets, tokens, phone numbers (CN mobile) and
  national ID numbers are never stored (`services/extractor.py`).
- **Opt-in gates:** extraction and injection both require the user to select this engine
  (`cognitive_engine`) and opt into memory; any profile error fails closed.
- **Owner scoping:** every query is filtered by `owner_type` + `owner_id`.
- **Admin-gated API:** all endpoints verify the JWT `is_admin` claim.
- **Controlled evolution:** prompts can only be modified for agents listed in
  `evolvable_agents`, and only after human approval via the dashboard.
- **Audit trail:** every proposal action is recorded in `improvement_audit`.

---

## Relation to memory_engine · 与 memory_engine 的关系

The two plugins are **two business directions, not competitors**:

| | memory_engine | cogevolution_substrate (Evolution Core) |
|---|---|---|
| Positioning | Enterprise business memory (knowledge base, commerce, finance) | Industrial / ML / data-flywheel evolution |
| Schema | memory_engine tables | isolated `cogevolution_substrate` schema |
| Engine gate | per-user `cognitive_engine` | per-user `cognitive_engine` |

A user picks the engine that fits their industry. Both may remain enabled; each only
activates for users who explicitly chose it, so coexistence is harmless.

---

## Known Limitations · 已知限制

- **Prompt reset on re-enable:** the framework reloads the agent prompt from
  `prompt_file` when the plugin is re-enabled, which resets `[EVOLUTION]` patches already
  applied to `system_prompt`. Re-applying is required after a re-enable.
- **Rule-based reflexion only:** `evolution_strategy` currently supports `rule` only; the
  `model` strategy is reserved.
- **`embedding` is best-effort:** when pgvector is unavailable, vector search silently
  degrades to keyword matching.
- **Legacy stubs:** `events.py`, `bindings/sim_telemetry.py`, `services/training.py` are
  non-active and kept for reference.

---

## Development & Tests · 开发与测试

```bash
# Unit tests (no database required)
python -m pytest plugins/cogevolution_substrate/tests/ -v
```

Covered scenarios:

- PII content is never written (`test_pii.py`).
- Short greetings are skipped; zero budget blocks writes.
- `content_hash` is isolated by owner and domain (normalized content still matches).
- Unselected / profile-failure users are never injected (fails closed).
- Injected memory block respects `max_memory_block_len`.

---

*This README reflects plugin v1.0.1. Generated from the actual source; if any behavior
differs from this document, the source is authoritative. · 本文档对应插件 v1.0.1，
以实际源码为准。*
