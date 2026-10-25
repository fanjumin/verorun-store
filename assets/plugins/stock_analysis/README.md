# Stock Analysis (`stock_analysis`)

> A-share multi-dimensional research: technical + fundamental + sentiment +
> AI-assisted deep research, batch analysis, morning brief, alerts, research
> DAG, paper broker. **Finance edition only.**

- **Identifier:** `stock_analysis`
- **Version:** 2.2.1
- **Category:** ai_agent
- **Menu group:** Analysis → Stock Analysis
- **Min app version:** 0.10.0
- **Compatible editions:** **finance only**
- **Agent role:** `stock_analyst`
- **Python deps:** pyyaml, pandas, requests, tushare>=1.4.0, akshare>=1.14.0
- **MCP Server:** `plugins/stock_analysis/tools/mcp_server.py` (stdio)

---

## Core Features

### 1. Multi-data-source
- `data_provider`: sina / tencent / akshare / tushare / fmp / polygon / wind / choice.
- `providers/base_v2.py` (19.9 KB), `providers/fmp_provider.py` (19.1 KB).

### 2. Technical / fundamental / sentiment
- `stock.technical_analysis`, `stock.fundamental_analysis`,
  `stock.sentiment_analysis`, `stock.signal_explanation` capabilities.
- `stock_skill.py` (48.1 KB).

### 3. Research DAG
- `research_dag.py` (19.2 KB).
- `auto_deep_research_on_batch=false`.

### 4. Evidence bundle
- `evidence_bundle.py` (22 KB).

### 5. Batch / morning brief / alerts
- Provided: `stock.alert.triggered`, `stock.batch.completed`,
  `stock.morning_brief.ready`.
- Listens: `scheduler.job_completed`.

### 6. Paper broker
- `broker/paper.py` (18.5 KB).

### 7. Gateway
- `gateway.py` (38.3 KB).

### 8. MCP
- AI agent fetches quotes / runs analysis via MCP.

### 9. Charts & intraday (since v2.2.0)
- **8 K-line periods**: `daily / weekly / monthly / 60m / 30m / 15m / 5m / 1m`;
  `1m` depends on the data source and returns **501** when unavailable (not 502),
  capped at 240 bars per request.
- **Three adjust bases**: `hfq / qfq / raw`; minute bars are re-based with daily
  factors. `meta` reports `price_basis / price_basis_declared / price_basis_adjusted /
  price_basis_anchor / adjust_unmet_reason`.
- **18 indicator keys + 13-row matrix** (`indicator_version=iv-5`): MA/MACD/KDJ/RSI/
  BOLL/volume-ratio plus ATR/OBV/DMI/CCI/WR/BIAS/ROC/EXPMA/SAR/VOL_MA.
- **Intraday `/api/intraday`**: 1m first, falls back to 5m; `days=1~5`; previous close
  not shifted; `avg=null` with a note when turnover is unavailable.
- **Paging**: `limit ≤ 2000` (1m ≤ 240); an out-of-range `offset` returns the
  **oldest available window** and sets `meta.offset_clamped_note` ("last page").
- **Per-user rate limiting**: key is `stock_analysis:<user_id>:<endpoint>`;
  a missing identity is rejected with 401 instead of sharing one bucket.

---

## At a glance

| Area | Value |
|---|---|
| Blueprint | `/admin/stock-analysis` (`routes.py`, 144.7 KB) |
| Models | `models_sa.py` (63.7 KB) |
| Skill | `stock_skill.py` (48.1 KB) |
| Gateway | `gateway.py` (38.3 KB) |
| Evidence | `evidence_bundle.py` (22 KB) |
| Providers | `providers/base_v2.py` (19.9 KB), `providers/fmp_provider.py` (19.1 KB) |
| DAG | `research_dag.py` (19.2 KB) |
| Paper broker | `broker/paper.py` (18.5 KB) |
| Python deps | pyyaml, pandas, requests, tushare, akshare |
| Permissions | network:request, routes, scheduler, dag, health |
| Capabilities | stock.technical_analysis, stock.fundamental_analysis, stock.sentiment_analysis, stock.signal_explanation, stock.chart_indicators |
| Hooks provided | stock.alert.triggered, stock.batch.completed, stock.morning_brief.ready |
| Hook listened | scheduler.job_completed |

### Config (7 items, sensitive keys password)

| Key | Default | Purpose |
|---|---|---|
| `tushare_token` | empty | Tushare token (empty = free akshare/sina) |
| `fmp_api_key` | empty | FMP (US fundamentals) |
| `polygon_api_key` | empty | Polygon (US quotes / news) |
| `data_provider` | empty | Force data source (empty = default order) |
| `data_cache_dir` | ./data/cache | Cache dir |
| `auto_deep_research_on_batch` | false | Auto deep research after batch |
| `index_selection` | [sh000001, sz399001, sz399006] | Overview indices |

---

## Five-minute quick start

1. **Finance edition** enable — install tushare / akshare.
2. Open **Analysis → Stock Analysis**.
3. Fill Tushare token (optional) / FMP / Polygon.
4. Input ticker → run technical / fundamental analysis.
5. Morning brief / alerts auto-run; AI agent calls via MCP.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | Investment research / quant | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Research | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
