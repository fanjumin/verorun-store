# AI Advisor (chatbot)

> English source of truth. Chinese mirror: `README.cn.md`. Both are kept in sync;
> last aligned version: **v1.7.0**.

## Overview

AI Advisor is the site-wide AI advisor / customer-service plugin for the VeroRun platform.
It ships **its own embeddable front-end widget** and **anonymous-capable public chat
endpoints**, so it can be injected into any page with a single `<script>` tag instead of
relying on markup hardcoded into a theme template.

Data lives in a dedicated PostgreSQL schema `chatbot` on the same instance: configuration,
threads and the agent registry are self-contained, while tickets and the knowledge base are
read cross-database from the main schema. All conversation logic is consolidated into one
engine, `service.answer()`, shared by five entry points: the web widget, Telegram, LINE,
Site Builder storefronts, and the admin console.

## Features

- **Multi-mode widget**: `bubble` (FAB + popover) / `drawer` / `inline` / `fullpage`,
  with drag + position memory, collapse/minimize, unread badge, SSE streaming typewriter,
  quick replies, source citations, CSAT stars, zh/en i18n, light/dark themes,
  Shadow DOM style isolation and keyboard accessibility.
- **Knowledge grounding (RAG)**: reuses the platform `agent_matrix.rag_retriever`
  (vector + keyword + RRF fusion), returns citations, and constrains the model to say so
  when the corpus does not cover a question. Retrieval failures degrade to no-context.
- **Server-side threads**: `chatbot_threads` / `chatbot_turns` persistence makes
  `max_history` actually enforced server-side; context survives page reloads.
- **Deterministic handoff**: keyword hit (word-boundary for ASCII, substring for CJK,
  longest-match first) or consecutive-failure threshold triggers handoff. Keyword hits
  skip the LLM entirely. Tickets are created automatically and `ticket_id` is written back.
- **Anonymous-capable**: HMAC-signed visitor tokens plus a sliding-window rate limiter,
  so chat, escalation and CSAT need no login.
- **Channels**: web widget, Telegram Bot webhook, LINE Messaging webhook — one engine,
  threads attributed by `channel`.
- **Operations console**: today overview, intent/sentiment distribution, hot topics,
  agent performance, QA check, Agent Copilot.
- **Single source of truth for config**: all reads/writes go through `config_store`
  (coercion, range validation, allow-list, all-or-nothing writes); admin changes take
  effect immediately in-process and site-wide within 30 s.

## Architecture

```
Any page                              Telegram / LINE
  one <script> tag                          webhook
        │                                      │
        ▼                                      ▼
  widget.js ──SSE──► /plugins/chatbot/*   /api/v1/channels/*
  (4 modes)            (public_api blueprint)  (channels/router)
                              │                  │
                              └───────┬──────────┘
                                      ▼
                          service.answer()  ← the only engine
                    ┌──────────┬──────┴──────┬────────────┐
                    ▼          ▼             ▼            ▼
              config_store  retrieval     threads     escalation
                    │          │             │            │
                    ▼          ▼             ▼            ▼
             chatbot schema  platform RAG  chatbot schema  main user_tickets
```

**Principles**

- **Reuse platform seams, zero core changes**: routes go through `register_routes()` →
  `mount_all_routes()` (auto-mounted in both admin and main_site; auto-404 when disabled
  via `_plugin_gatekeeper`); cross-module calls go through
  `shared/plugin_access.PLUGIN_TOUCHPOINTS`; RAG via `agent_matrix.rag_retriever`;
  DDL via the plugin's own `migrations/`.
- **Own schema, main DB read-only** (the single write is `user_tickets` on escalation).
- **One engine, many entries**: no channel may assemble its own prompt or decide handoff.

## Directory layout

```
chatbot/
├── README.md / README.cn.md / CHANGELOG.md
├── plugin.json                # metadata, permissions, settings_schema, metadata.agents
├── __init__.py                # setup / activate / register_routes / hook registration
├── config_store.py            # runtime config single source of truth
├── threads.py                 # server-side threads + handoff rules
├── retrieval.py               # RAG grounding + robust JSON parsing
├── guard.py                   # visitor tokens, rate limit, webhook verify, PII masking
├── service.py                 # unified conversation engine
├── escalation.py              # [TICKET_CREATE] parsing + ticket creation
├── public_api.py              # public blueprint /plugins/chatbot/*
├── routes.py                  # admin API + channel webhooks
├── models.py / stats.py       # DB & tables / analytics, QA, copilot
├── channels/router.py         # Telegram / LINE transport
├── prompts/sub_chatbot_prompt.md
├── migrations/                # versioned SQL documents
├── templates/admin_chatbot.html
├── templates/widget/          # chatbot-widget.js / chatbot-widget.css
├── i18n/                      # zh-CN.yml / en.yml
└── tests/                     # offline unit tests (193 Python + 26 node)
```

## Install

Requirements: VeroRun ≥ 0.10.0, PostgreSQL, a working LLM provider; for RAG,
`plugins/_base/embeddings` and pgvector (missing → automatic keyword-only degradation).

1. Place `chatbot/` under `plugins/` and make sure `plugin.json` has `enabled: true`.
2. **Set `CHATBOT_VISITOR_SECRET`.** Mandatory for multi-process deployments: admin,
   main_site and site_builder are separate processes, and a token issued by one will not
   verify in another without a shared secret.
3. Restart. The plugin creates the schema and all tables (including
   `chatbot_threads` / `chatbot_turns`), seeds defaults, idempotently migrates legacy data
   from the main DB, absorbs config left behind in the main DB, registers agent
   capabilities and two filter hooks, and mounts three blueprints.
4. Configure copy, widget mode, RAG, rate limits and handoff rules under
   “AI & Content” → “AI Advisor”.

Channels (optional): configure Telegram / LINE credentials in the IM Gateway plugin and
**always** set `TELEGRAM_SECRET_TOKEN` / `LINE_CHANNEL_SECRET`. Without them the webhooks
return 403 (fail-closed) rather than silently accepting traffic.

## Front-end integration

One line on any page (`GET /plugins/chatbot/embed` returns the snippet for current config):

```html
<script src="/plugins/chatbot/widget.js" defer
        data-mode="bubble" data-position="bottom-right" data-theme="auto"></script>
```

- Manual control: add `data-autoboot="false"`, then call
  `VeroRunAdvisor.init({ mode: 'drawer', container: '#help', lang: 'en' })`.
- Runtime API: `init / boot / setMode / toggle / clear / destroy`; instances live in
  `VeroRunAdvisor.instances`.
- `inline` mode takes `data-container="#selector"`; `fullpage` suits a dedicated route.
- The widget depends on no host globals or styles; Shadow DOM mounting isolates CSS.

## Configuration

The single source of truth is `chatbot.plugin_configs`. Admin writes are mirrored into
`plugin_registry.config` for store/metadata purposes, but runtime code never reads the mirror.

| Key | Type | Default | Notes |
|---|---|---|---|
| `enabled` | bool | true | Master switch; public endpoints return 503 and routes 404 when off |
| `auto_escalate` | bool | true | Create a ticket on handoff |
| `title` / `subtitle` | string | AI Advisor | Widget header |
| `welcome_message` / `help_hint` | string | — | Greeting and hint |
| `avatar_url` | string | "" | Avatar (media library picker + upload in admin) |
| `agent_id` | string | chat_assistant | Bound `agent_registry` entry (**honoured since v1.7.0**) |
| `max_history` | int 1–50 | 20 | Server-side retained rounds |
| `float_button_text` | string | AI Advisor | FAB label |
| `handoff_keywords` | JSON array | ~19 zh/en terms | ASCII word-boundary, CJK substring, longest-first |
| `handoff_max_fails` | int 1–10 | 3 | Consecutive-failure threshold |
| `widget_mode` | enum | bubble | bubble / drawer / inline / fullpage |
| `widget_position` | enum | bottom-right | bottom-right / bottom-left |
| `widget_theme` | enum | auto | auto / light / dark |
| `quick_replies` | JSON array | [] | Up to 4 chips shown |
| `embed_key` | string | "" | Optional shared key for public endpoints; empty = open |
| `rag_enabled` | bool | true | Enable knowledge grounding |
| `rag_top_k` | int 1–20 | 5 | Matches the platform retriever cap |
| `rag_min_score` | float 0–1 | 0.0 | Fusion-score floor |
| `rate_limit_per_min` | int 1–600 | 10 | Per visitor per minute |
| `visitor_token_secret` | string | "" | Empty → uses `CHATBOT_VISITOR_SECRET` |

Any invalid key (out of range, wrong type, not allow-listed) rejects the **whole batch** —
configuration never ends up half-updated.

## API

### Public (anonymous-capable, prefix `/plugins/chatbot`)

| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/config` | none (optional `embed_key`) | Boot config; **field allow-list**, no ops rules or secrets |
| POST | `/session` | none | Issues visitor token + server-side `thread_id` (6/min/IP) |
| POST | `/messages` | visitor token or JWT | `stream` defaults to true (SSE); false returns JSON |
| POST | `/actions/escalate` | visitor token or JWT | Handoff ticket |
| POST | `/actions/csat` | visitor token or JWT | Rating 1–5 |
| GET | `/widget.js`, `/widget.css` | none | Assets with ETag / 304 / nosniff |
| GET | `/embed` | none | Embed snippet (`?format=text` for plain text) |

SSE events: `start` → `token`×N → `done` (with `reply`, `thread_id`, `handoff`,
`ticket_id`, `references`); `error` (with `code`) on failure. The internal
`[TICKET_CREATE]` marker and ticket JSON are suppressed from the stream.

### Admin (prefix `/admin/chatbot`)

| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/chat` | admin | Unified engine; keeps the `data.reply` contract |
| GET / POST | `/settings` | admin | Full runtime config |
| GET / POST | `/handoff_rules` | admin | Same source as chat config |
| GET | `/stats`, `/hot_topics`, `/agent_performance` | admin | Console data |
| POST | `/log_session`, `/qa_check`, `/copilot_suggest`, `/csat`, `/escalate` | login | Kept for existing callers |

### Channel webhooks (prefix `/api/v1/channels`)

| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/telegram/webhook` | `X-Telegram-Bot-Api-Secret-Token`, **fail-closed** | 403 when secret unset |
| POST | `/line/webhook` | `x-line-signature` (HMAC-SHA256), **fail-closed** | Same |

## Dependencies and platform seams

| Dependency | Purpose |
|---|---|
| `plugins._base.db` | Pooled connection to the `chatbot` schema |
| `agent_matrix.engine` / `intent` / `rag_retriever` / `models` | LLM, intent & sentiment, RAG, capability registration |
| `plugin_manager.hooks` / `base` / `logger` | Hooks, lifecycle, logging |
| `shared.plugin_access` | The only sanctioned core→plugin entry (6 touchpoints registered) |
| `auth-center.models` / `services.jwt_service` | Main-DB tickets, login parsing (imported **at call time**) |
| `plugins.im_gateway` | Telegram / LINE credentials |

Provided hooks (filters; first callback argument is `value`):

| Hook | Semantics |
|---|---|
| `chatbot/config` | pass a dict of overrides → get merged runtime config |
| `chatbot/chat` | pass a message string or `{'message': ...}` → get the `service.answer()` result dict |

Agent registration: name `Advisor Agent`, identifier `chat_assistant`, role `sub`,
domain `chatbot`, capabilities `chatbot.faq` / `chatbot.ticket` / `chatbot.human_handoff`,
prompt loaded from `prompts/sub_chatbot_prompt.md`, driven by `metadata.agents`
in `plugin.json`.

## Known limitations

- Analytics are “today”-scoped: no date-range filter, no export; `trend` is computed but
  not yet charted in the console.
- No live agent workspace: `copilot_suggest` only proposes replies, tickets are created
  with `assigned_to = NULL`, and there is no skills-based routing, queueing or SLA.
- Channels are Telegram and LINE only; no WhatsApp / WeChat / WeCom; identity is not
  unified across channels.
- No multimodal input (image/voice/file), no proactive campaigns, no A/B testing or
  offline evaluation set.
- Rate limiting is a **per-process** sliding window: with N processes the effective limit
  is roughly `rate_limit_per_min × N`. Use a gateway or Redis for a global budget.
- `POST /api/v1/chat` in main_site (anonymous, own prompt assembly) remains a separate
  implementation; this release only corrected where it reads handoff rules. Migrating it
  to the `chatbot/chat` hook or the `service.answer` touchpoint is recommended next.

## Development

```bash
# Python offline unit tests (193; DB / LLM / main DB / outbound HTTP all stubbed)
python -m unittest discover -s plugins/chatbot/tests -t .

# Widget pure-logic tests (26; node >= 18, no jsdom needed)
node --check plugins/chatbot/templates/widget/chatbot-widget.js
node --test plugins/chatbot/tests/widget/core.test.js
```

Conventions: a new config key must be added to `config_store.RUNTIME_KEYS` / `DEFAULTS`
**and** to `plugin.json` `config` / `settings_schema`; a new table must be added to the
idempotent DDL in `models.py` / `threads.py` **and** to a versioned file in `migrations/`.

## License

Part of the VeroRun platform, under the platform's unified license agreement.
