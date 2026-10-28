# 1688 Supply Chain (`ali_api`)

> 1688.com product sourcing. Keyword / image search, AI-rewritten titles
> and descriptions, multi-image management, one-click publish into the local
> VeroRun shop, OAuth token management, Redis cache, per-operator rate
> limiting, and purchase-order tracking.

- **Identifier:** `ali_api`
- **Version:** 2.5.6
- **Category:** supply_chain
- **Menu group:** Business Center → 1688 Supply Chain
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `business`
- **Python deps:** `openai`, `redis`

---

## Core Features

### 1. 1688 OpenAPI client
- `services/alibaba_client.py` (16.1 KB) — legacy OpenAPI client.
- `services/alibaba_client_v2.py` (9.7 KB) — v2 product detail
  (`/v2/products/<id>`).
- Default gateway `https://gw.open.1688.com/openapi`, configurable.
- Full OAuth flow: `/oauth/url` → `/oauth/callback` → `/oauth/status` →
  `/oauth/refresh` → `/oauth/disconnect`.

### 2. Product search & collection
- **Keyword search**: `POST /items/search` (line 477).
- **Image search**: `services/image_search_service.py` (5.3 KB).
- **Collect**: `POST /items/collect` (line 339) — pulls a 1688 item into
  `ali_api_items`.
- **Product detail**: `GET /v2/products/<product_id>` (line 1699) — v2 API.

### 3. AI copywriting
- `services/ai_processor.py` (16.9 KB) — calls OpenAI to rewrite title /
  description.
- `POST /items/<id>/ai-optimize` (line 566) — one-shot full optimization.
- `POST /items/<id>/ai-titles` (line 629) — generate candidate titles.
- `POST /items/<id>/select-title` (line 675) — pick a candidate.
- Agent prompt: `agents/supply_chain_prompt.md`.

### 4. Image management
- `GET /items/<id>/images` (line 892)
- `POST /items/<id>/images/upload` (line 927) — upload local image to
  replace 1688 image.
- `DELETE /items/<id>/images/<idx>` (line 1004)
- `POST /items/<id>/images/reorder` (line 1052)

### 5. Publish to local shop
- `POST /items/<id>/publish` (line 707) — one-click publish to VeroRun
  local `shop` plugin.
- `POST /items/<id>/unpublish` (line 851) — unpublish.

### 6. Purchase orders
- `GET /orders/supplier` (line 1784) — supplier orders.
- `POST /orders/create-purchase` (line 1839) — place purchase order.
- `POST /orders/sync-tracking` (line 1945) — sync tracking.

### 7. Cache & rate limit
- `services/cache_service.py` (13.2 KB) — Redis cache for 1688 responses.
- `services/rate_limiter.py` (15.9 KB) — per-operator rate limit.
- `GET /cache/stats`, `POST /cache/clear`, `GET /rate-limit/stats`.

### 8. Review & logs
- `services/review_service.py` (4.8 KB) — product review.
- `GET /logs` (line 1167) — API call log.
- `ali_api_logs` table + `ali_api_user_stats` table.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `ali_api`, **10 tables** (see below) |
| Blueprint | `/admin/ali-api` (`routes/admin.py`, 76.9 KB) |
| Admin endpoints | **34** (see below) |
| Frontend | `templates/admin_ali_api.html` (63 KB) + `templates/ali_admin/index.html` (22 KB) + `static/ali_console.js` (36 KB) |
| Python deps | `openai`, `redis` |
| Permissions | `network:request`, `api:write` |
| Capabilities | `product.search`, `product.rewrite`, `product.publish` |
| Dashboard | `total_items`, `published_today`, `api_calls_24h` |

### 10 tables

| Table | Purpose |
|---|---|
| `ali_api_items` | Collected items (1688 raw fields + local optimized copy) |
| `ali_api_reviews` | Product review records |
| `ali_api_logs` | 1688 OpenAPI call log |
| `ali_api_user_stats` | Per-operator stats |
| `ali_api_tokens` | OAuth token (encrypted) |
| `ali_oauth_states` | OAuth CSRF state |
| `ali_api_config` | Plugin config (gateway URL, etc.) |
| `ali_purchase_orders` | Purchase orders |
| `agent_registry` | Agent registry |
| `schema_meta` | Schema metadata |

### 34 HTTP endpoints (grouped)

**Pages / static**
- `GET /`, `GET /dashboard`, `GET /items`, `GET /items/<id>`
- `GET /uploads/<path:filename>`, `GET /csrf-token`

**Search / collect**
- `POST /items/search`
- `POST /items/collect`
- `GET /v2/products/<product_id>`

**AI optimization**
- `POST /items/<id>/ai-optimize`
- `POST /items/<id>/ai-titles`
- `POST /items/<id>/select-title`

**Images**
- `GET /items/<id>/images`
- `POST /items/<id>/images/upload`
- `DELETE /items/<id>/images/<idx>`
- `POST /items/<id>/images/reorder`

**Publish**
- `POST /items/<id>/publish`
- `POST /items/<id>/unpublish`

**Purchase orders**
- `GET /orders/supplier`
- `POST /orders/create-purchase`
- `POST /orders/sync-tracking`

**OAuth**
- `GET /oauth/url`, `GET|POST /oauth/callback`, `GET /oauth/status`
- `POST /oauth/refresh`, `POST /oauth/disconnect`

**Ops**
- `GET /cache/stats`, `POST /cache/clear`
- `GET /rate-limit/stats`
- `GET /logs`
- `GET /config` / `POST /config`
- `GET /settings` / `POST /settings`

---

## Five-minute quick start

1. Enable — runs migration `v0.0.0_to_v2.1.0.sql` (10 tables).
2. Open **Business Center → 1688 Supply Chain**.
3. Click **OAuth authorize** → redirect to 1688 → callback →
   `/oauth/status` shows connected.
4. **Search** → **Collect** → item lands in list.
5. **AI optimize** → pick title → manage images → **Publish** to local shop.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / integrator | `README.md` |
| Technical whitepaper | Backend / e-commerce engineers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Merchandisers / ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
