# Payment Gateway (`payment`)

> Unified multi-channel payment gateway. Alipay / WeChat / Stripe / PayPal /
> Lianlian, credentials stored in an independent PG schema, hook-driven
> charge / callback / refund.

- **Identifier:** `payment`
- **Version:** 1.7.0
- **Category:** shop
- **Menu group:** Business Center → Payment Config
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `finance`
- **Python deps:** `stripe`

---

## Core Features

### 1. 5 channel adapters
- `gateways/wechat.py` (22.5 KB) — WeChat Pay.
- `gateways/alipay.py` (16.4 KB) — Alipay.
- `gateways/paypal.py` (11.6 KB) — PayPal.
- `gateways/lianlian.py` (12.3 KB) — Lianlian.
- Stripe (`stripe` SDK).

### 2. Config management (`routes/admin.py`, 11.6 KB)
- `GET /providers` — list channels.
- `GET/POST /configs/<provider>`, `PUT/DELETE /configs/<provider>`.
- `GET /check/<provider>` — connectivity check.

### 3. Hooks
- Provided: `payment/create`, `payment/verify_notify`, `payment/refund`,
  `payment/charge`.
- Order / subscription plugins initiate payment via hooks.

### 4. Dashboard
- `total_logs`, `total_configs`, `recent_payments` (24h).

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `payment` (independent schema) |
| Blueprint | `/admin/payment` (`routes/admin.py`, 11.6 KB) |
| HTTP endpoints | **6** (admin) |
| Channel adapters | wechat.py (22.5 KB), alipay.py (16.4 KB), lianlian.py (12.3 KB), paypal.py (11.6 KB), Stripe |
| Admin UI | `templates/admin_payment.html` (9.1 KB) |
| Python deps | `stripe` |
| Permissions | api:read/write, admin:access |
| Capabilities | payment.create, payment.refund, payment.query, payment.gateway.manage |
| Hooks provided | payment/create, payment/verify_notify, payment/refund, payment/charge |
| Dashboard | total_logs / total_configs / recent_payments |

### Config (1 item)

| Key | Default | Purpose |
|---|---|---|
| `default_currency` | CNY | Default order currency |

---

## Five-minute quick start

1. Enable — auto-creates tables.
2. Open **Business Center → Payment Config**.
3. Fill Alipay / WeChat / Stripe / PayPal credentials.
4. `GET /check/<provider>` to verify connectivity.
5. Order plugin initiates via `payment/create` hook.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | E-commerce ops / finance | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops / finance | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
