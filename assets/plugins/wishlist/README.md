# Wishlist (`wishlist`)

> User product favorites / wishlist. Toggle, paginated list, batch status
> check, cross-device sync.

- **Identifier:** `wishlist`
- **Version:** 1.7.4
- **Category:** shop
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `business`
- **Python deps:** none

---

## Core Features

### 1. Add / remove
- `wishlist.add` / `wishlist.remove` capabilities.

### 2. List
- `wishlist.list` capability.
- `max_page_size=50` (up to 200).

### 3. Cross-device sync
- `wishlist.sync` capability.
- Provides `wishlist/sync` hook.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `wishlist` (`models.py`, 1.5 KB) |
| Python deps | none |
| Permissions | api:read, user:profile |
| Capabilities | wishlist.add, wishlist.remove, wishlist.list, wishlist.sync |
| Hook provided | wishlist/sync |
| Dashboard | total_favorites / active_users |

### Config (1 item)

| Key | Default | Purpose |
|---|---|---|
| `max_page_size` | 50 | Max page size (1–200) |

---

## Five-minute quick start

1. Enable — auto-creates tables.
2. Add "❤ favorite" button on product page.
3. User views list at /wishlist.
4. Auto-syncs across devices.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | E-commerce ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Users | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
