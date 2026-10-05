# AI Relay Station (`ai_relay`)

OpenAI-compatible AI model relay plugin for VeroRun: multi-provider channel
routing, per-token authentication, credit billing, rate limiting, and an
admin console — all inside one plugin schema (`ai_relay`) without touching
the system core.

- Version: **1.4.1**
- External contract: OpenAI REST/SSE only (`/v1/*`)
- Storage: dedicated PostgreSQL schema, shared connection pool
  (`plugins/_base/db`), no standalone database

[中文文档](./README.zh-CN.md)

## Features

- **OpenAI-compatible gateway**: chat/completions (stream + non-stream),
  models, embeddings, images/generations, rerank, audio/transcriptions.
- **Protocol adapters**: OpenAI-compatible, Anthropic, Gemini — upstream
  responses and SSE streams are normalized to the OpenAI shape; clients
  always see one contract.
- **Multi-channel routing**: priority tiers, region affinity (`cn`/`os`),
  weighted random within a tier, automatic retry on the next channel for
  upstream 429/5xx/timeouts.
- **Key pool per channel**: weighted key pickup, health score, cooldown and
  circuit breaker (PostgreSQL authoritative + in-process fast path).
- **Token authentication**: `sk-vr-` tokens stored as SHA-256 only, optional
  IP whitelist, short-lived auth cache, per-IP brute-force protection.
- **Per-token credit quota (1.4.0)**: `quota_credits` caps the cumulative
  `cost_credits` a single token can consume (`-1` = unlimited, the default);
  usage aggregates live from `relay_usage_logs` (index `ix_relay_usage_token`)
  and an over-budget token is rejected with 402.
- **Three-layer authorization**: token allow-list × user group allow-list ×
  industry-plan entitlements (union semantics).
- **Dual-unit billing**: credits (charged to users) + fen (upstream cost),
  BIGINT everywhere, priced **per 1M tokens** (sub-fen precision since 1.3.0),
  atomic credit freeze before forwarding and guaranteed settlement in `finally`
  (stream disconnects included).
- **Rate limiting**: sliding-window RPM/TPM/concurrency at token/user/group/
  ip scope, plus **channel-scope aggregate limits (1.2.0)** that throttle all
  users sharing one upstream account before traffic reaches the provider.
- **Metering fallback chain**: upstream usage → tiktoken → heuristic.
- **Finance operations**: recharge orders with idempotent payment callbacks,
  manual credit grants, ledger, usage logs, daily aggregation, stale-freeze
  reconciliation, monthly plan grants.
- **User self-service**: end users can check balance, create recharge orders
  and manage their own API tokens from the user center via
  `/plugin/ai_relay/api/*` (valid user JWT only, no admin rights).
- **SSRF protection**: scheme allow-list, full DNS resolution checks with
  IPv4-mapped IPv6 normalization, reuses `net_proxy` block lists when
  available, fails closed; private upstreams require explicit opt-in.
- **Upstream credential encryption**: Fernet derived from `ENCRYPTION_KEY`,
  fail-closed (a relay that cannot encrypt keys will not store or send them).

## Request lifecycle

```
client ──> auth (token + brute-force guard)
        ──> authorize (token / group / industry plan)
        ──> rate limit (token/user/group/ip sliding window)
        ──> freeze credits (single atomic UPDATE)
        ──> pick channel ── SSRF validation ── channel gate (1.2.0)
        ──> pick key ──> forward (retry ≤ max_retry_per_request channels)
        ──> finally: settle (ledger + usage log + daily stats + freeze state)
```

## Directory layout

- [`routes_gateway.py`](./routes_gateway.py) — external `/v1/*` blueprint
- [`routes_admin.py`](./routes_admin.py) — admin/user API under
  `/plugin/ai_relay/admin/*` and `/api/*`
- [`models.py`](./models.py) — schema lifecycle (advisory-lock migrations)
  and shared queries
- [`relay/`](./relay) — `auth`, `authorize`, `ratelimit`,
  `channel_ratelimit`, `balance`, `router`, `keys`, `upstream`, `metering`,
  `credits`, `settle`, `recharge`, `entitlements`, `benchmark`
- [`adapters/`](./adapters) — `openai_compat`, `anthropic`, `gemini`
- [`migrations/`](./migrations) — `v1.0.0` init (19 tables),
  `v1.1.0` observability, `v1.2.0` channel rate limiting,
  `v1.3.0` per-1M pricing precision, `v1.4.0` per-token quota
- [`scheduler.py`](./scheduler.py) — idempotent APScheduler jobs
- [`templates/relay_admin.html`](./templates/relay_admin.html) — admin panel
  partial (loaded via the admin dashboard)
- [`i18n/`](./i18n) — `en.yml` (identity) + `zh-CN.yml`, identical key sets

## Requirements

- VeroRun host application (`min_app_version: 0.10.0`)
- Python packages: `httpx`, `tiktoken` (accurate token counts); optional:
  `anthropic` (native Anthropic adapter)
- Environment variable **`ENCRYPTION_KEY`** (>= 16 characters). Upstream API
  keys cannot be added until it is set.

Migrations run automatically on plugin activation (`activate()`), guarded by
a transaction advisory lock and a `schema_migrations` ledger; every migration
is idempotent (`IF NOT EXISTS`).

## Configuration

Manifest `config` keys (editable via plugin settings):

| Key | Default | Meaning |
|---|---|---|
| `upstream_connect_timeout` | 10 | Upstream connect timeout (seconds) |
| `upstream_read_timeout` | 300 | Upstream read timeout, covers long SSE connections |
| `max_retry_per_request` | 2 | Max channel retries per request |
| `allow_private_upstream` | false | Allow channels pointing at private/reserved networks |
| `log_prompt_preview` | false | Store a short prompt preview in usage logs |
| `credits_per_fen` | 10 | Recharge exchange rate: 1 fen = N credits |
| `ref_speed_tpm` | 3600 | Reference speed for the pricing factor |
| `speed_alpha_pct` | 50 | Congestion exponent (basis points) |
| `speed_factor_min_bp` | 5000 | Pricing factor floor |
| `speed_factor_max_bp` | 20000 | Pricing factor ceiling |
| `speed_factor_mode` | congestion | `congestion` or `compensation` |

## Channel aggregate limits (1.2.0)

One upstream account is shared by every external user, so per-user limits
cannot protect the provider. Configure a per-channel ceiling:

- Admin panel: **Relay Channels → channel row → Limits**
- API:

```http
GET    /plugin/ai_relay/admin/channels/{id}/limits
PUT    /plugin/ai_relay/admin/channels/{id}/limits
DELETE /plugin/ai_relay/admin/channels/{id}/limits
```

```json
{ "rpm": 600, "tpm": 200000, "concurrent": 8 }
```

`0` (or no row) means unlimited. When a channel is over its ceiling the
request is retried on the next channel; if every channel is exhausted the
client receives `502 upstream_error`. Gate failures fail open (billing stays
fail-closed).

## Scheduled jobs

| Job | Schedule | Purpose |
|---|---|---|
| `freeze_reconcile` | every 10 min | Settle/release stale frozen requests |
| `daily_backfill` | 03:30 | Rebuild previous-day daily stats |
| `rate_events_cleanup` | 04:00 | Delete rate events older than 7 days |
| `usage_logs_cleanup` | 04:30 | Delete usage logs past the region retention window (CN 180d / OS 30d) |
| `entitlement_expire` | 01:30 | Expire passed grants |
| `orphan_cleanup` | 02:00 | Remove rows of deleted users |
| `monthly_grant` | 1st 01:00 | Industry-plan monthly credit grants |
| `speed_benchmark` | Mon 05:00 | Weekly speed measurement and price recompute |

All jobs are idempotent (atomic UPDATE / unique UPSERT) and safe under
multiple workers.

## Security notes

- Tokens are stored as SHA-256; plaintext is shown once at creation.
- Upstream keys are Fernet-encrypted at rest and never logged.
- Upstream error bodies are sanitized (base URL/host removed, `sk-` masked,
  truncated) before being returned to clients.
- Admin endpoints require a JWT with `is_admin`; user endpoints require a
  valid user JWT. There is no fail-open path.
- SSRF validation applies to every selected channel on every request, not
  only at configuration time.

## Current limitations (1.5.0)

- Legacy `*_per_1k` pricing columns are retained but frozen for rollback
  safety; billing, admin and `/v1/models` all use the `*_per_1m` columns
  (new-preferred, legacy ×1000 fallback during the transition).
- The speed benchmark enumerates candidates and recomputes prices, but actual
  measurement (`_measure_one`) is not implemented yet; the speed factor stays
  1.0 until then.
- SSRF is validated before connecting, but the HTTP client does not yet pin
  the verified IP at connection time (DNS rebinding hardening is planned).
- Rate-limit check/insert is not transaction-serialized and fails open by
  design; only billing is fail-closed.
- Sensitive admin operations are logged to the application log; a structured
  audit-trail table is planned.

## Changelog

- **1.5.0** — regional compliance convergence (CN vs. OS). The deployment-level
  ceiling resolves from `VR_PROFILE` (falling back to `APP_REGION`); a
  token-level `region_policy` can only tighten below it, never escalate. Under
  CN the relay is limited to `chat`/`embedding`, upstream channels are limited
  to the CN region and region-agnostic (`any`) channels, inbound content safety
  is enabled (fail-closed keyword checker, extensible via the
  `ai_relay_content_check` filter), and usage logs
  are retained 180 days instead of 30. Compliance rejections answer
  `403 model_not_allowed` / `400 content_policy_violation` **before** quota
  freeze, so a blocked request never freezes balance and leaves no usage log.
  New admin export `GET /admin/compliance/export` bundles registration/audit
  material with no secrets and no wordlist. Per-token quota becomes a rolling
  window aligned with the retention period.
- **1.4.1** — defects reported by a third-party test run against a live
  install: non-retryable upstream 4xx now answer with the real status code
  and the upstream `error.type`/`error.message` instead of collapsing into
  `502 upstream_error`; key health/cooldown and channel fuse updates use the
  PostgreSQL scalar functions `LEAST`/`GREATEST` (`MIN`/`MAX` are aggregates
  there) so both statements commit in one transaction; the per-token quota
  check counts the current request's estimate, so the request that would
  exceed the limit is rejected with 402 instead of only the next one;
  upstream hosts that fail to resolve are rejected fail-closed.
- **1.4.0** — per-token credit quota (migration `v1.4.0`): `relay_tokens`
  gains `quota_credits` (`-1` = unlimited), enforced at the gateway with a
  402 response and aggregated live from `relay_usage_logs` via the new
  `ix_relay_usage_token` index; new user self-service API
  (`/plugin/ai_relay/api/balance`, `/api/recharge`, `/api/tokens`) with the
  user-center entry surfaced dynamically by plugin enablement.
- **1.3.0** — per-1M pricing precision (migration `v1.3.0`): sub-fen market
  rates now bill correctly (e.g. CNY 4/1M = 400 fen/1M); `/v1/models` exposes
  a `pricing` block in fen per 1M tokens; freeze estimate, router, admin API
  and UI all use the new columns with legacy-key fallback; legacy columns
  retained but frozen.
- **1.2.0** — per-channel RPM/TPM/concurrency aggregate gate
  (`relay_channel_limits`, migration `v1.2.0`), admin endpoints and UI.
- **1.1.0** — observability: cached-token counts and TTFT.
- **1.0.0** — initial release: gateway, adapters, routing, billing,
  entitlements, recharge, scheduler (19 tables).
