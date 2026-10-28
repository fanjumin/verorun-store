# AI Relay Station (`ai_relay`)

> OpenAI-compatible AI model gateway. Multi-channel routing (OpenAI /
> Anthropic / Gemini / OpenAI-compatible), per-user API tokens, credit
> metering & billing, rate limits, content safety, region policy,
> speed-tier pricing, and observability.

- **Identifier:** `ai_relay`
- **Version:** 1.5.1
- **Category:** ai_agent
- **Menu group:** Business Center → Relay Channels / API Tokens / Usage & Billing
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro
- **Agent role:** `ops`
- **Depends on:** `payment >= 1.0.0`
- **Python deps:** `httpx`, `tiktoken`, `cryptography`; optional `anthropic`

---

## Core Features

`ai_relay` is the platform's LLM gateway. Clients hit `/v1/...` with a
VeroRun-issued token; the plugin authenticates, freezes credits, routes to a
healthy upstream channel, streams the response, and settles the bill.

### 1. Multi-channel routing (3 adapters)
- `adapters/openai_compat.py` — any OpenAI-compatible upstream.
- `adapters/anthropic.py` (11.9 KB) — Claude native protocol (SSE).
- `adapters/gemini.py` (8.2 KB) — Google Gemini.
- `relay/router.py` (4.8 KB) — health + weighted routing.
- `relay/channel_ratelimit.py` (3.9 KB) — channel-level RPM/TPM.

### 2. Six OpenAI-compatible endpoints
- `GET /v1/models`
- `POST /v1/chat/completions` (incl. SSE streaming)
- `POST /v1/embeddings`
- `POST /v1/images/generations`
- `POST /v1/rerank`
- `POST /v1/audio/transcriptions` (≤25 MB)

### 3. Two-phase credit metering
- **Pre-request freeze**: `relay/metering.py` + `relay/credits.py`,
  estimated at `est_max_output=4096`; recorded in
  `relay_requests_freeze`.
- **Post-request settle**: `relay/settle.py` (10.8 KB) reconciles by
  real token count.
- **Negative-balance trip**: `negative_trip_credits` threshold disables
  the offending token immediately.
- **Recharge**: `relay/recharge.py` (9 KB), `credits_per_fen=10`
  (1 fen = 10 credits), routed through `payment`.

### 4. Security & compliance
- **Content safety**: `relay/content_safety.py` (6.6 KB) +
  `content_words.json` keyword filter.
- **Region policy**: `relay/region_policy.py` (8.4 KB, v1.5.0) —
  allow/deny models and channels by region.
- **Auth**: `relay/auth.py` (6.8 KB) + `relay/keys.py` (9.3 KB) —
  token issue / verify / rotate.
- **IP trust**: `trusted_proxy_hops` controls `X-Forwarded-For` depth.
- **Private upstream audit**: `allow_private_upstream` default false.

### 5. Entitlements & quotas
- `relay/authorize.py` — token-level model whitelist.
- `relay/entitlements.py` (6.6 KB) — industry plans + bindings.
- `relay/ratelimit.py` (7.1 KB) — per-user/channel RPM/TPM.

### 6. Speed-tier pricing
- `relay_price_rules` table + `speed_factor_mode`:
  - `congestion` (default): slower models cost more.
  - `compensation`: slower models cost less.
- `speed_factor_min_bp=5000` / `max_bp=20000` (basis points).
- `relay/benchmark.py` (8.7 KB) + `scheduler.py` (16.1 KB) run
  weekly speed benchmarks paid by `benchmark_user_id`.

### 7. Observability
- `relay_usage_logs` per-request, `relay_usage_daily` daily rollup.
- `relay_ledger` credit ledger.
- Dashboard: 24h requests / 24h tokens / 24h credits / 7d upstream
  cost (fen) / active tokens.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `ai_relay`, **20 tables** (see below) |
| Blueprints | `/v1` gateway (`routes_gateway.py`, 38.7 KB) + `/admin/relay` (`routes_admin.py`, 76.4 KB) |
| Admin endpoints | **~55** (channels / models / groups / tokens / plans / billing) |
| Gateway endpoints | 6 OpenAI-compatible |
| Internal modules | 19 .py under `relay/` (auth, authorize, balance, benchmark, channel_ratelimit, content_safety, credits, entitlements, keys, metering, ratelimit, recharge, region_policy, router, settle, upstream, …) |
| Python deps | `httpx`, `tiktoken`, `cryptography`; optional `anthropic` |
| Permissions | `network:request`, `api:read`, `api:write`, `admin:access`, `routes`, `scheduler` |
| Dashboard | 5 stats (see above) |

### 20 tables

| Table | Purpose |
|---|---|
| `relay_channels` | Upstream channels (OpenAI / Anthropic / Gemini / compat) |
| `relay_channel_keys` | Per-channel multi-key (rotation / health) |
| `relay_models` | Model catalog (price, context, speed tier) |
| `relay_model_bindings` | Model ↔ channel bindings |
| `relay_groups` | Token quota groups |
| `relay_tokens` | Issued API tokens |
| `relay_accounts` | User credit accounts |
| `relay_ledger` | Credit ledger |
| `relay_usage_logs` | Per-request log |
| `relay_usage_daily` | Daily rollup |
| `relay_rate_rules` | Rate-limit rules |
| `relay_rate_events` | Rate-limit events |
| `relay_price_rules` | Speed-tier pricing |
| `relay_requests_freeze` | Pre-request freeze records |
| `relay_recharge_orders` | Recharge orders |
| `relay_grant_records` | Admin grants |
| `relay_industry_plans` | Industry plans |
| `relay_plan_bindings` | User ↔ plan bindings |
| `relay_entitlements` | User entitlements |
| `relay_channel_limits` | Channel RPM/TPM caps (v1.2.0) |

### 11 capabilities

`relay.chat`, `relay.embeddings`, `relay.rerank`, `relay.image`,
`relay.audio`, `relay.channel.manage`, `relay.token.manage`,
`relay.quota.query`, `relay.usage.report`, `relay.rate_limit`,
`relay.entitlement.check`.

### Key config (21 items, common subset)

| Key | Default | Purpose |
|---|---|---|
| `upstream_read_timeout` | 300 | Upstream read timeout (SSE long conn, seconds) |
| `max_retry_per_request` | 2 | Max retries per request |
| `est_max_output` | 4096 | Freeze estimate when client omits max_tokens |
| `credits_per_fen` | 10 | 1 fen = N credits |
| `max_recharge_fen` | 1000000 | Per-order recharge cap (fen, = 10k yuan) |
| `max_audio_upload_bytes` | 26214400 | Audio upload cap (25 MB) |
| `max_images_per_request` | 10 | Images per /v1/images/generations |
| `negative_trip_credits` | 0 | Negative-balance trip threshold (0=off) |
| `trusted_proxy_hops` | 0 | Reverse-proxy hop count (IP parsing) |
| `allow_private_upstream` | false | Allow private-network upstream (audited) |
| `log_prompt_preview` | false | Store prompt preview in logs (default hash-only) |
| `speed_factor_mode` | congestion | congestion / compensation |
| `benchmark_user_id` | 0 | Speed-benchmark payer account (0=off) |

---

## Five-minute quick start

1. Enable — runs migrations v1.0.0 → v1.5.0 (6 SQL files).
2. Open **Business Center → Relay Channels**.
3. Add an upstream channel (e.g. OpenAI) with its key.
4. **API Tokens** → issue a token for a user.
5. Point the OpenAI client at `https://<host>/v1` with that token.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / integrator | `README.md` |
| Technical whitepaper | Backend / platform engineers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops / finance | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
