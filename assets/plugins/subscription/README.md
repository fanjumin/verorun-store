# Subscription (`subscription`)

> Unified pay-as-you-go subscription marketplace. Per-item SKU billing
> with dual-environment payment routing (CN: Alipay / WeChat; INTL:
> Stripe / PayPal). Trial, grace period, auto-renew, invoices, deploy codes.

- **Identifier:** `subscription`
- **Version:** 2.6.4
- **Category:** system
- **Menu group:** Business Center → Subscription
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `finance`
- **Python deps:** `defusedxml` (parse Alipay / WeChat callbacks)

---

## Core Features

### 1. SKU / items
- `sub_items` table.
- `GET /api/items`, `GET /api/my`, `GET /api/check/<item_key>`.
- Admin: `GET/POST /admin/items`, `DELETE /admin/items/<key>`.

### 2. Subscription lifecycle
- `POST /api/subscribe`, `POST /api/cancel`, `POST /api/renew`.
- `user_subscriptions` table.
- `trial_days=0`, `grace_days=3`, `auto_renew_default=true`.

### 3. Orders
- `sub_orders` table.
- `GET /api/orders`, `POST /api/orders/<no>/retry`.
- Admin: `GET /admin/orders`, `POST /admin/orders/<no>/refund`.

### 4. Payment callbacks (4 channels)
- `POST /api/notify/alipay`, `/wechat`, `/stripe`, `/paypal`.
- `sub_payment_events` table.

### 5. Invoices
- `sub_invoices` table.
- `GET /api/my/invoices`, `GET /api/my/invoices/<no>/download`.

### 6. Deploy codes
- `deploy_codes` table.
- Admin: `GET/POST /admin/deploy-codes`, `POST /admin/deploy-codes/<code>`.

### 7. Audit
- `sub_audit_log` table.

### 8. Admin
- `GET /admin/users`, `GET /admin/stats`, `GET /admin/billing`, `GET /admin/`.
- Portal: `GET /portal`.

### 9. Hooks (7)
- Provided: `subscription/has`, `subscription/list`, `subscription/subscribe`,
  `subscription/cancel`, `subscription/renew`,
  `subscription/check_module_access`, `subscription/site_plans`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `subscription`, **7 tables** |
| Blueprint | `/subscription` (`routes.py`, 20.8 KB) |
| HTTP endpoints | **29** |
| Service | `services.py` (54 KB) |
| Migration | `migrate_legacy.py` (12.2 KB) |
| Templates | subscribe.html (24.6 KB), subscribe_admin.html (23.6 KB) |
| Python deps | defusedxml |
| Permissions | api:read/write, admin:access |
| Capabilities | subscription.list, subscription.subscribe, subscription.cancel, subscription.renew, subscription.check_access |
| Hooks provided | 7 (see above) |
| Dashboard | active_subscriptions / total_plans / total_revenue_fen |

### 7 tables

| Table | Purpose |
|---|---|
| `sub_items` | Subscription SKUs |
| `user_subscriptions` | User subscriptions |
| `sub_orders` | Orders |
| `sub_payment_events` | Payment callbacks |
| `sub_audit_log` | Audit log |
| `deploy_codes` | Deploy codes |
| `sub_invoices` | Invoices |

### Config (3 items)

| Key | Default | Purpose |
|---|---|---|
| `trial_days` | 0 | Trial days (0 = none) |
| `grace_days` | 3 | Grace period after expiry |
| `auto_renew_default` | true | Default auto-renew for new |

---

## Five-minute quick start

1. Enable — auto-creates 7 tables.
2. Open **Business Center → Subscription**.
3. Add SKUs in Items.
4. Users subscribe via `/portal` → Alipay / WeChat / Stripe / PayPal.
5. Invoices auto-issued.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Finance / ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
