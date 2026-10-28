# Mini App Builder (`mini_app_builder`)

> Natural-language & template mini-program generation, multi-platform code
> output (WeChat / Douyin / Telegram / H5), developer-account management,
> preview / download / deploy. Merges dev_accounts.

- **Identifier:** `mini_app_builder`
- **Version:** 2.8.5
- **Category:** tools
- **Menu group:** System → Mini Apps
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, minipro
- **Agent role:** `builder`
- **Depends on:** `oauth_config>=1.0.0`
- **Replaces:** `dev_accounts`
- **Python deps:** none

---

## Core Features

### 1. AI generation (`mini_app/generators/base.py`, 24.1 KB)
- `POST /mini-app/generate` — natural language / template.
- `ai_generation_enabled=true`.
- `max_concurrent_tasks=5`, `task_ttl_seconds=3600`.
- `GET /mini-app/status/<task_id>`.

### 2. Multi-platform output
- Platforms: WeChat, Douyin, Telegram Mini App, H5.
- `GET /mini-app/versions/<id>/download/<platform>`.
- `GET /mini-app/download/<platform>/<task_id>`.
- `POST /mini-app/deploy/<platform>`.

### 3. Project & version management
- `GET/POST /mini-app/projects`, `GET/DELETE /projects/<id>`.
- `GET /projects/<id>/versions`.
- `output_base=workspace/mini_apps`.

### 4. Developer-account management (merged dev_accounts)
- `platform_users.py` — per-platform developer accounts (AppID / Secret).
- `encryption_key_env=MINI_APP_ENCRYPTION_KEY` — credentials encrypted.
- `GET /mini-app/platforms`, `PUT /mini-app/platforms/<platform>`.
- Hooks: `dev_accounts.created` / `updated` / `deleted`.

### 5. Mini-program public API (`public_api.py`, 38.5 KB)
- **Chat**: `POST /chat/stream`, `POST /chat/send`, `GET /chat/history`,
  `POST /chat/escalate`.
- **Knowledge**: `GET /knowledge/search`.
- **Site**: `GET /site/info`, `/site/pages`, `/site/page/<slug>`.
- **User**: `GET /user/profile`.
- **Shop**: `GET /shop/products`, `/shop/products/<id>`,
  `/shop/products/<id>/skus`, `/shop/recommend`, `/shop/cart`,
  `/shop/cart/add|update|remove`, `/shop/checkout`, `/shop/orders`,
  `/shop/orders/<oid>/cancel`, `/shop/pay/<oid>`,
  `/shop/pay/<oid>/stub-confirm`, `/shop/pay/status/<oid>`.

### 6. Publish submodule (`submodules/publish/routes.py`, 15.7 KB)
- Mini-program login: `routes_mini_login.py` (15.4 KB).

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `mini_app_builder`, **3 tables** + platform user tables |
| Blueprints | `/admin/site-builder/mini-app` (`routes.py`, 42 KB) + `/api/mini-program` (`public_api.py`, 38.5 KB) |
| HTTP endpoints | **30+** (12 admin + 22 public) |
| Generator | `mini_app/generators/base.py` (24 KB) |
| Admin UI | mini_apps.html (30.7 KB) + admin_devaccounts.html (17.5 KB) |
| Python deps | none |
| Permissions | admin:access, database:write, network:request, file:write, crypto:encrypt |
| Capabilities | mini_app.generate, mini_app.deploy, mini_app.preview |
| Hooks provided | mini_app.generate/deploy/preview, dev_accounts.created/updated/deleted |
| Hooks listened | site.created, brand.updated |
| Dashboard | total_accounts, active_accounts, total_projects, total_versions |

### 3 tables

| Table | Purpose |
|---|---|
| `mini_app_projects` | Projects |
| `mini_app_versions` | Versions |
| `mini_app_sessions` | Generation task sessions |

### Config (5 items)

| Key | Default | Purpose |
|---|---|---|
| `output_base` | workspace/mini_apps | Generated code output dir |
| `task_ttl_seconds` | 3600 | Task TTL |
| `max_concurrent_tasks` | 5 | Max concurrent tasks |
| `ai_generation_enabled` | true | AI generation switch |
| `encryption_key_env` | MINI_APP_ENCRYPTION_KEY | Credential encryption key env |

---

## Five-minute quick start

1. Enable — auto-creates tables, depends on oauth_config.
2. Set env `MINI_APP_ENCRYPTION_KEY`.
3. Open **System → Mini Apps**, configure platform developer accounts.
4. Describe requirement in natural language → `POST /generate`.
5. Pick platform to download / deploy.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Developers / ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
