# Order Notifications (`order_notify`)

> Event-driven automatic notifications for the whole order lifecycle:
> created, paid, shipped, refunded. Sends both in-app message and email.

- **Identifier:** `order_notify`
- **Version:** 1.5.4
- **Category:** shop
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `finance`
- **Python deps:** none

---

## Core Features

### 1. 4 order events listened
- `order/created` — placed.
- `order/paid` — payment success.
- `order/shipped` — shipped.
- `order/refunded` — refunded.

### 2. Dual-channel
- **In-app**: via `notification.send` permission.
- **Email**: via `email.send` permission (depends on email plugin).

### 3. Notification log
- `notification_log` table.
- Dashboard: `total_notifications`, `today_notifications`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `order_notify`, **1 table** |
| Trigger | Hook-driven (no HTTP endpoints) |
| Python deps | none |
| Permissions | notification.send, email.send |
| Capabilities | order.notify.send, order.notify.track |
| Hooks listened | order/created, order/paid, order/shipped, order/refunded |
| Dashboard | total_notifications, today_notifications |

### 1 table

| Table | Purpose |
|---|---|
| `notification_log` | Notification log (order / event / channel / time) |

### Config
No `settings_schema` — templates are in code.

---

## Five-minute quick start

1. Enable — auto-creates 1 table.
2. Ensure email plugin SMTP is configured.
3. Order / payment / ship / refund auto-triggers notification.
4. Dashboard shows today's notification count.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | E-commerce ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
