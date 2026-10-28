# Currency Converter (`currency_converter`)

> Multi-currency display with real-time exchange rates (Frankfurter primary
> / ER-API fallback), GeoIP auto-detection, per-user preference, automatic
> price conversion, and a frontend widget.

- **Identifier:** `currency_converter`
- **Version:** 1.5.4
- **Category:** shop
- **Menu group:** Business Center → Currency Settings
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `business`
- **Python deps:** `httpx`

---

## Core Features

### 1. Real-time rate sync (`services.py`, 14.4 KB)
- **Primary**: `https://api.frankfurter.dev/v1/latest`.
- **Fallback**: `https://open.er-api.com/v6/latest`.
- `refresh_interval_minutes=60`.
- `cache_ttl_minutes=60`.
- 15 currencies enabled by default: CNY / USD / EUR / JPY / GBP / HKD /
  KRW / AUD / CAD / SGD / THB / MYR / PHP / IDR / VND.

### 2. Rates table (`exchange_rates`)
- Latest rates (base CNY).
- `last_sync` dashboard stat.

### 3. User preference (`user_currency_prefs`)
- Per-user preferred currency.
- Listens to `user/login`, `user/registered` — new users defaulted via
  GeoIP.

### 4. GeoIP auto-detection
- `enable_geoip=true`.
- `GET /geoip` returns suggested currency.

### 5. 11 HTTP endpoints
- **Public**: `GET /`, `GET /rates`, `POST /convert`,
  `GET/POST /preference`, `GET /geoip`.
- **Admin**: `POST /manage/sync` (manual), `GET /manage/stats`,
  `GET /manage/config`.
- **Settings**: `GET/POST /settings`.

### 6. Frontend widget
- `static/currency_widget.js` (8.8 KB) — currency switcher.
- Prices auto-convert per user preference after embedding.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `currency_converter`, **2 tables** |
| Blueprint | `/api/currency` (`routes.py`, 11.2 KB) |
| HTTP endpoints | **11** |
| Service layer | `services.py` (14.4 KB) |
| Frontend | `static/currency_widget.js` (8.8 KB) |
| Admin UI | `templates/admin_currency.html` (7.6 KB) |
| Python deps | `httpx` |
| Permissions | `api:read`, `user:profile`, `network:request` |
| Capabilities | `currency.convert`, `currency.rates`, `currency.preference` |
| Hooks provided | `currency/convert`, `currency/rates`, `currency/preference` |
| Hooks listened | `user/login`, `user/registered` |
| Dashboard | `currency_rates`, `last_sync` |

### 2 tables

| Table | Purpose |
|---|---|
| `exchange_rates` | Latest rates (base CNY, multi-currency) |
| `user_currency_prefs` | Per-user preferred currency |

### Key config

| Key | Default | Purpose |
|---|---|---|
| `base_currency` | CNY | System base currency (DB storage) |
| `default_currency` | CNY | New-visitor default display |
| `refresh_interval_minutes` | 60 | Sync interval (≥15) |
| `enabled_currencies` | CNY,USD,EUR,JPY,GBP,HKD,KRW,AUD,CAD,SGD | Enabled currencies (comma-separated) |
| `enable_geoip` | true | GeoIP auto-detect |

---

## Five-minute quick start

1. Enable — auto-creates 2 tables, first rate sync.
2. Open **Business Center → Currency Settings**.
3. Tick currencies to enable.
4. Embed `currency_widget.js`.
5. Visitor prices auto-convert by GeoIP / preference.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | End user / integrator | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
