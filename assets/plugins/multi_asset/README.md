# Multi-Asset Data (`multi_asset`)

> Futures / options / funds / bonds market data. CFI-aligned (ISO
> 10962:2021) instrument typing, exchange double-field modelling, and a
> governance-ready storage layer that tags every bar with license,
> security level, origin and dataset lineage. Reuses the stock_analysis
> provider contract and data gateway.

- **Identifier:** `multi_asset`
- **Version:** 1.2.2
- **Category:** ai_agent
- **Menu group:** Analysis → Multi-Asset
- **Min app version:** 0.10.0
- **Compatible editions:** finance
- **Agent role:** `stock_analyst`
- **Depends on:** `stock_analysis>=2.0.1`
- **Python deps:** pyyaml, pandas, akshare>=1.14.0, tushare>=1.4.0

---

## Core Features

### 1. Four asset classes
- Futures, options, funds, bonds.
- CFI (ISO 10962:2021) instrument typing.

### 2. Provider adapters
- `adapters/providers.py` (17.4 KB) — unified contract.
- `adapters/tushare_ma.py` (16.3 KB) — Tushare adapter.
- `data_provider`: empty / akshare / sina / sa_gateway.

### 3. Symbol modelling (`asset_symbol.py`, 23.1 KB)
- Exchange double-field.

### 4. Data link (`data_link.py`, 13.6 KB)
- Reuses stock_analysis data gateway.

### 5. Governance & lineage
- Every bar tagged with license, security_level, origin, dataset lineage.
- `ma_license`, `ma_dataset_registry` tables.
- `ma_quality_report`.

### 6. PIT (Point-in-Time)
- `ma_pit_fundamentals`, `ma_pit_consensus`: bitemporal read/write; feed path `pit_feed.py` (`POST /api/pit/feed`, via the stock_analysis gateway).
- `ma_index_membership` (read/write implemented; no feed path yet).
- Quality rules R1-R3 write `ma_quality_report` (`POST /api/quality/run`); read via `GET /api/quality/report`.

### 7. Hooks
- Provided: `asset.data.ready`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `multi_asset`, **13 tables** |
| Blueprint | `/admin/multi-asset` (`routes.py`, 12.7 KB) |
| Models | `models.py` (43.9 KB) |
| Symbols | `asset_symbol.py` (23.1 KB) |
| Providers | providers.py (17.4 KB) + tushare_ma.py (16.3 KB) |
| Data link | `data_link.py` (13.6 KB) |
| Admin UI | `templates/multi_asset.html` (10 KB) |
| Python deps | pyyaml, pandas, akshare, tushare |
| Permissions | network:request, routes, scheduler, health, events |
| Capabilities | asset.futures/options/funds/bonds, asset.data_fetch, asset.storage, asset.provenance, data.pit, data.quality |
| Hook provided | asset.data.ready |

### 13 tables

| Table | Purpose |
|---|---|
| `ma_bars` | Bars (with license / security level / origin / lineage) |
| `ma_fund_ref` | Fund reference |
| `ma_bond_ref` | Bond reference |
| `ma_future_contracts` | Futures contracts |
| `ma_option_contracts` | Option contracts |
| `ma_option_greeks` | Option Greeks |
| `ma_fetch_log` | Fetch log |
| `ma_license` | License tags |
| `ma_dataset_registry` | Dataset registry |
| `ma_pit_fundamentals` | PIT fundamentals |
| `ma_pit_consensus` | PIT consensus |
| `ma_index_membership` | Index membership |
| `ma_quality_report` | Data quality report |

### Config (1 item)

| Key | Default | Purpose |
|---|---|---|
| `data_provider` | empty | Source (empty / akshare / sina / sa_gateway) |

---

## Five-minute quick start

1. Enable (finance edition) — auto-creates 13 tables, depends on stock_analysis.
2. Open **Analysis → Multi-Asset**.
3. Set `data_provider` (optional).
4. Pick a futures / option / fund / bond contract to view bars.
5. Quality report shows lineage and license.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Quant / finance engineers | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| Role integration | Analysis agents | [`docs/role-integration.md`](docs/role-integration.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
