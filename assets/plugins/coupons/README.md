# Smart Coupons (`coupons`)

> Scene-aware coupon engine. Fixed / percentage / first-month discounts,
> per-user limits, product / plan applicability, stackable / non-stackable,
> AI recommendation, subscription integration.

- **Identifier:** `coupons`
- **Version:** 1.5.4
- **Category:** shop
- **Menu group:** Business Center → Coupon Management
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `business`
- **Python deps:** none

---

## Core Features

### 1. Coupon engine (`engine.py`, 19.2 KB)
- Types: `fixed`, `percent`, `first_month_only` (subscription).
- Thresholds: `min_amount`, `min_quantity`.
- Limits: `usage_limit` (total), `per_user_limit` (per user).
- Window: `active_from` / `active_to` / `expire_at`.
- Scope: `applicable_products`, `applicable_plans`, `scene`.
- Stacking: `stackable=0/1`.

### 2. Admin endpoints (7)
- `GET /admin/list`
- `POST /admin/create`
- `POST /admin/update/<cid>`
- `POST /admin/delete/<cid>` (soft delete)
- `GET /admin/stats`
- `POST /admin/distribute` — targeted issuance
- `GET /admin/redemptions/<cid>`

### 3. User endpoints (5)
- `POST /validate` — validate before checkout
- `GET /user/list` — my coupons
- `POST /user/available` — available coupons
- `POST /apply` — apply to order
- `GET /scenes` — scene enum

### 4. AI recommendation
- `POST /ai/recommend` — best coupon for user profile / cart.
- `coupon.recommend` capability.

### 5. Bridge endpoints (3, for subscription / order plugins)
- `GET /bridge/list`, `GET /bridge/stats`,
  `GET /bridge/redemptions/<cid>`.

### 6. Hooks
- Provided: `coupon/validate`, `coupon/apply`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `coupons`, **3 tables** |
| Blueprint | `/api/coupons` (`routes.py`, 12.7 KB) |
| HTTP endpoints | **16** (7 admin + 5 user + 1 AI + 3 bridge) |
| Core engine | `engine.py` (19.2 KB) |
| Admin UI | `templates/admin_coupons.html` (16 KB) |
| Python deps | none |
| Permissions | `order.read`, `order.write`, `user.read`, `admin:access` |
| Capabilities | `coupon.create`, `coupon.validate`, `coupon.analyze`, `coupon.recommend` |
| Hooks provided | `coupon/validate`, `coupon/apply` |
| Dashboard | `total_coupons`, `active_coupons`, `total_used`, `total_discount` |

### 3 tables

| Table | Purpose |
|---|---|
| `coupons` | Coupon definition (type / value / threshold / limit / scope / stackable) |
| `coupon_redemptions` | Redemption log (coupon / user / order / discount in fen) |
| `schema_meta` | Schema version |

### Config
No `settings_schema` — all parameters live on the coupon definition.

---

## Five-minute quick start

1. Enable — auto-creates 3 tables.
2. Open **Business Center → Coupon Management**.
3. Create a fixed coupon (e.g. ¥10, threshold ¥100).
4. User enters code at checkout → `POST /validate` → `POST /apply`.
5. Admin views redemptions and total saved amount.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / integrator | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
