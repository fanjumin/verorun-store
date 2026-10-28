# AI Advisor (`chatbot`)

> Site-wide AI customer-service chatbot. Embeddable widget (bubble /
> drawer / inline / fullpage), public chat API, RAG grounding on the
> platform knowledge base, server-side multi-turn threads, deterministic
> handoff to a human ticket, and Telegram / LINE webhooks.

- **Identifier:** `chatbot`
- **Version:** 1.7.4
- **Category:** ai_agent
- **Menu group:** AI & Content → AI Advisor
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, minipro
- **Agent role:** `service`
- **Python deps:** none

---

## Core Features

### 1. Embeddable widget (4 modes)
- `templates/widget/chatbot-widget.js` (34.6 KB) + `.css` (13 KB).
- 4 modes: `bubble` (corner bubble, default), `drawer`, `inline`,
  `fullpage`.
- 2 positions: `bottom-right` / `bottom-left`.
- 3 themes: `auto` / `light` / `dark`.
- Served publicly at `GET /widget.js`, `/widget.css`, `/embed`.
- Optional `embed_key` to protect public endpoints.

### 2. Public chat API (8 `/widget` endpoints)
- `GET /config` — widget config (title, welcome, quick replies).
- `POST /session` — create session (visitor token).
- `POST /messages` — send message, SSE stream response.
- `POST /actions/escalate` — user-initiated handoff.
- `POST /actions/csat` — satisfaction rating.
- `GET /widget.js`, `widget.css`, `embed` — static assets.

### 3. Server-side multi-turn threads
- `threads.py` (17.2 KB) — `chatbot_threads` + `chatbot_turns`.
- Keeps last `max_history=20` turns per session.
- `service.py` (16.8 KB) — orchestrates retrieval + LLM stream + guardrails.

### 4. RAG grounding (main knowledge base)
- `retrieval.py` reads cross-schema from the main KB (no copy).
- `rag_enabled=true`, `rag_top_k=5`, `rag_min_score=0.0`.

### 5. Deterministic handoff
- **Keyword trigger**: 18 built-in CN/EN keywords (`human / agent / 客服 /
  转人工 / 投诉 / 商务 ...`), configurable via `handoff_keywords`.
- **Failure trigger**: `handoff_max_fails=3` consecutive misses.
- `auto_escalate=true` auto-creates a ticket.
- `escalation.py` emits `chatbot/ticket` hook.

### 6. Visitor profile
- `profile.py` (14.4 KB) — per-visitor memory injected at prompt time.
- `chatbot_webhook_seen` table idempotently dedupes webhook events.

### 7. IM webhooks
- `POST /webhook/telegram/webhook`, `/webhook/line/webhook` — let
  Telegram / LINE bots drive the same pipeline.

### 8. Admin endpoints (16)
- `POST /chat` — backend simulate chat.
- `GET/POST /settings`
- `POST /log_session`, `GET /stats`, `GET /hot_topics`,
  `GET /agent_performance`
- `POST /qa_check`, `POST /copilot_suggest`
- `POST /csat`, `POST /escalate`
- `GET/POST /handoff_rules`

### 9. Sub-agent
- Built-in `chat_assistant` (role_type=sub, domain=chatbot),
  prompt `prompts/sub_chatbot_prompt.md`.
- 3 capabilities: `chatbot.faq`, `chatbot.ticket`,
  `chatbot.human_handoff`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `chatbot`, **7 tables** |
| Blueprints | `/admin/chatbot` (`routes.py`, 21 KB) + `/widget` (`public_api.py`, 17 KB) + webhook sub-blueprints |
| HTTP endpoints | **24** (8 public + 16 admin/webhook) |
| Widget | `chatbot-widget.js` (34.6 KB) + `.css` (13 KB) |
| Admin UI | `templates/admin_chatbot.html` (17 KB) |
| Python deps | none |
| Permissions | `api:read`, `network:request`, `routes`, `admin:access` |
| Hooks provided | `chatbot/config`, `chatbot/chat` |
| Capabilities | `chatbot.faq`, `chatbot.ticket`, `chatbot.human_handoff` |
| Dashboard | `today_sessions`, `handoff_rate`, `avg_csat` |

### 7 tables

| Table | Purpose |
|---|---|
| `schema_meta` | Schema metadata |
| `plugin_configs` | Config (title, welcome, etc.) |
| `agent_registry` | Sub-agent registry |
| `chatbot_sessions` | Sessions |
| `chatbot_threads` | Multi-turn threads |
| `chatbot_turns` | Per-turn messages |
| `chatbot_webhook_seen` | Webhook idempotency |

### Key config (23 items, common subset)

| Key | Default | Purpose |
|---|---|---|
| `enabled` | true | Master switch |
| `auto_escalate` | true | Auto-create ticket on handoff |
| `title` / `subtitle` | AI Advisor / Powered by AI Engine | Window title |
| `welcome_message` | Hello! ... | Welcome text |
| `agent_id` | chat_assistant | Bound sub-agent |
| `max_history` | 20 | Turns kept (1–50) |
| `handoff_keywords` | 18 CN/EN keywords | Handoff trigger |
| `handoff_max_fails` | 3 | Consecutive fails before handoff |
| `widget_mode` | bubble | bubble/drawer/inline/fullpage |
| `widget_position` | bottom-right | bottom-right/left |
| `widget_theme` | auto | auto/light/dark |
| `embed_key` | empty | Public endpoint key |
| `rag_enabled` | true | RAG grounding |
| `rag_top_k` | 5 | RAG recall (1–20) |
| `rag_min_score` | 0.0 | RAG min similarity |
| `rate_limit_per_min` | 10 | Per-visitor RPM |
| `visitor_token_secret` | empty | Visitor token secret (multi-process) |

---

## Five-minute quick start

1. Enable — auto-creates 7 tables.
2. Open **AI & Content → AI Advisor**.
3. Set title / welcome; pick widget mode.
4. Optionally set `embed_key` to protect public endpoints.
5. Add the widget snippet to your site — bubble appears in the corner.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / integrator | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops / support | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
