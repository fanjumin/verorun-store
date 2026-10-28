# Butterfly Engine

**小扰动，大分叉 — fragility · scenario simulation · descriptive early-warning, fully evidence-linked and replayable**

Butterfly Engine is VeroRun's standalone **market-fragility risk plugin**. The Finance Edition is the native use case; a **cross-domain pilot** (sentiment / intelligence / competitive series, visible to the Research and Enterprise Editions) extends the same primitives to non-finance equal-spaced series via the state channel. It measures how "fragile" a series is to small perturbations, simulates conditioned historical shock scenarios, and produces **descriptive** early-warning levels — with a mandatory evidence chain and full replayability for audit.

> ⚠️ **"Butterfly effect" is a brand metaphor, not a scientific claim.** The method's real name is **conditional-analog dispersion** — the dispersion of conditionally matched historical continuations when the anchor window is epsilon-perturbed. It is **not** dynamical-sensitivity quantification, **not** network-contagion modeling, **not** causal inference, and **not** forecasting. It produces **no trading signals and no investment advice**.

---

## Fixed boundary statement

These statements are injected by the render layer (admin page and Skill outputs). They are part of the product contract and must not be removed, rewritten, or bypassed:

| # | Boundary |
|---|-----------|
| 1 | Conditional-analog dispersion, **not** dynamical sensitivity. |
| 2 | No cross-institution / cross-market **network contagion** is modeled. |
| 3 | Association under historical conditions, **not** causation. |
| 4 | Tail distribution is **not identifiable** with the current sample size (when triggered). |
| 5 | This result is a **risk measurement, not investment advice**. |
| 6 | Scenario simulation, **not** a forecast. |
| 7 | `alert`-level conclusions require **human confirmation** before any client delivery. |

---

## Core capabilities (four primitives)

| Primitive | MCP tool | HTTP endpoint | Summary |
|-----------|----------|---------------|---------|
| `butterfly.sensitivity` | `butterfly_estimate_sensitivity` | `POST /plugin/butterfly-engine/sensitivity` | Conditional-analog dispersion: volatility-standardized kNN analog continuation, dispersion curve `D(h)`, curve-shape self-check, fragility/sensitivity proxy score. |
| `butterfly.amplification` | `butterfly_amplification_path` | `POST /plugin/butterfly-engine/amplification` | Conditioned historical shock-response amplification intervals + modal fork scenarios (kernel-density modes, Dirichlet-shrunk probabilities). |
| `butterfly.early_warning` | `butterfly_detect_early_warning` | `POST /plugin/butterfly-engine/early-warning` | GARCH(1,1)-filtered composite early-warning indicator (AR1 + variance + drift z-composite vs. bootstrap p95) → `info / watch / alert`. |
| `butterfly.evidence_link` | `butterfly_link_evidence` | `POST /plugin/butterfly-engine/evidence-link` | Link an evidence-bundle snapshot (PII-gated); echoes the LLM egress permission for downstream gates. |

---

## Quick start

### Requirements

- VeroRun **Finance / Research / Enterprise Edition** (`compatible_editions: ["finance", "research", "enterprise"]`; the finance path is native, the cross-domain pilot serves research/enterprise)
- App version **≥ 0.63.0**
- Python ≥ 3.12 with `numpy` (the engine is pure numpy; no scipy dependency)

### Install & enable

1. Install the plugin through the kernel plugin manager. The PF-10 migrations run automatically and create **6 tables** under the `butterfly_engine` schema.
2. Enable the plugin; the admin route, MCP server and scheduler hooks become available (the `scheduler` permission is declared — required by the kernel for enablement).

### Admin page

`/admin/butterfly` — SSO-guarded. Provides:

- Dashboard statistics (8 counters/gauges wired to real persistence, see below);
- Run history with row-level ownership filtering;
- Per-run **replay** (recomputation and field-by-field comparison);
- **Pending alert confirmation queue** (`confirm` / `reject`).

### MCP tools

```bash
python plugins/butterfly_engine/tools/mcp_server.py --list
```

Exposes: `butterfly_estimate_sensitivity`, `butterfly_amplification_path`, `butterfly_detect_early_warning`, `butterfly_link_evidence` (stdio JSON-RPC 2.0).

### Capability API

Internal-service token (`X-Internal-Token`, when `INTERNAL_SERVICE_TOKEN` is set) or admin SSO. All responses use the unified envelope:

```json
{ "success": true, "data": { ... }, "trace": { ... }, "error": null }
```

`trace` is mandatory audit information.

---

## How it works (one page)

1. **Sensitivity** — `estimate_sensitivity`
   - Validate series & institutional flags → quality gate (R6).
   - Volatility-standardize returns.
   - kNN conditional replay: the anchor window's `k0` nearest historical analog segments; their continuations form the ensemble.
   - Dispersion curve `D(h)` (cross-ensemble std of cumulative responses); saturation-aware log fit → `curve_shape` (`exponential_like` / `flat` / `noisy`).
   - `epsilon` drives **only** the rank-stability probe (`perturbation_rank_flips`) and `params_hash` — the primary curve is invariant to epsilon. This is **not** a dynamical Lyapunov exponent.
   - Output ceiling: *"fragility/sensitivity proxy elevated under historical conditions"* — association, never prediction.

2. **Amplification** — `amplification_path`
   - Shock events are conditionally replayed: same volatility regime tercile **and** same direction (direction = the event's own magnitude sign).
   - Effective sample recount; fixed relaxation order on shortage (direction → vol bucket); fail-closed below the floor (30).
   - Honest tails: p5/p95 only when ≥ 100 events (`tail_status = identifiable`).
   - Modal fork scenarios via kernel-density histogram modes + Dirichlet(1+n) probability shrinkage + bootstrap consistency gate (≥ 0.7; downgrade to 2 modes; still unstable → refuse).
   - `conduction_weights` are **diagnostic-only echoes** until M2 backtest calibration.

3. **Early warning** — `detect_early_warning`
   - GARCH(1,1) MLE (pure numpy; convergence ≤ 1e-6 relative LL, max 500 iterations, 5 fixed dispersed starts — fully deterministic).
   - Standardized residuals → AR1 / rolling variance / drift signals → z-scores → weighted composite (0.4/0.4/0.2).
   - Trigger needs the composite to exceed **both** the threshold z **and** the bootstrap(1000) baseline p95.
   - Levels: `info` (record) / `watch` (trim suggestion) / `alert` (escalate to human confirmation queue; **never** auto-delivered to clients).

4. **Evidence** — `link_evidence`
   - Evidence-bundle snapshot persisted with PII gate (recursive key-name + value-pattern scan), license status and `llm_egress_allowed` echo.

---

## Configuration

All settings are tunable via the admin settings schema (`plugin.json → settings_schema`); defaults below.

| Setting | Default | Range / Floor | Semantics |
|---------|---------|---------------|-----------|
| `default_window` | 60 | 20 – 500 | Anchor/analog window length (trading days). |
| `min_data_points` | 30 | ≥ 30 | Fail-closed below this (schema floor = audit threshold; the sensitivity primitive itself requires ~119 points for window+h). |
| `min_shock_events` | 30 | ≥ 30 | Fail-closed below this (audit floor v3.1 L2). |
| `min_tail_events` | 100 | ≥ 100 | Below this `tail_status = not_identifiable` (audit floor v3.1 L2). |
| `perturbation_epsilon` | 0.01 | 0.0001 – 0.1 | Drives **only** the rank-stability probe + `params_hash`; the primary dispersion curve is invariant to it. |
| `ensemble_size` | 200 | 20 – 1000 | kNN ensemble size (k0 = min(ensemble, candidates)). |
| `conduction_weights` | leverage 0.34 / herding 0.33 / algo 0.33 | JSON | Persisted per run; backtest-calibrated registry (`butterfly_conduction_weights`); diagnostic-only until calibrated. |
| `warning_threshold_z` | 1.96 | 1.5 – 4.0 | Below 1.96 → significant-degradation note + trace record + human confirmation (floor 1.5, v3.1 L2). |
| `bootstrap_iterations` | 1000 | 200 – 10000 | Bootstrap baseline for the early-warning p95 gate. |
| `require_evidence` | true | — | Reject runs without an evidence trace. |
| `require_data_quality` | true | — | Reject series failing the quality gate (audit R6). |
| `pii_guard` | true | — | PII scan before storing evidence snapshots. |
| `warning_cooldown_hours` | 6 | ≥ 0 | DB-authoritative warning cooldown (multi-worker safe). |
| `max_history_points` | 2500 | ≥ 100 | History subsampled above this for the kNN index (audit R13); subsample derived from the run seed (L3). |
| `max_input_points` | 200000 | ≥ 100 | Hard input length bound — above this `INPUT_TOO_LARGE` (413), DoS guard (audit F7). |
| `retention_days` | 730 | ≥ 30 | Run-history retention. |
| `max_runs_per_symbol` | 0 | ≥ 0 | Unbounded by default. |

---

## Data contract & A-share institutional handling

Butterfly Engine **never touches the network** — series are always assembled and validated by the data-provider Skill (`butterfly-data-prepare`).

- **Quality gate (R6)**: missing-value rate, halt/suspension segments, limit-up/down truncation, adjustment basis, length/frequency consistency, look-ahead bias — any failure blocks computation.
- **`limit_flags[]`**: `limit_up`, `limit_down`, `halt`, `ex_dividend`, `t_plus_1`, `short_constrained`, `one_word_board`.
- **Limit clipping**: returns clipped at **±10%** (uniform A-share main-board proxy). ST (±5%) and ChiNext/STAR (±20%) bands are **not** differentiated; the board note is written to `trace.institutional.limit_clip.board_note`.
- **Halts**: flat/halt segments (price series with >20% unchanged consecutive closes) are fail-closed rejected; halt-flagged segments are marked in the trace.
- **Adjustment**: providers must deliver **point-in-time (PIT) adjusted** series (`ex_dividend` flags are recorded in the trace only).
- **Calendar**: align to the exchange trading calendar; the horizon `h` counts trading days.
- **Size**: hard cap `max_input_points` (default 200,000) → `INPUT_TOO_LARGE`.

---

## Cross-domain series (pilot, 1.1.0)

The same primitives measure non-finance equal-spaced series (opinion indices, intelligence-flow scores, competitive-event series) through the **state channel**: pass `input_kind="state"` and the engine skips the price→return conversion and all A-share institutional handling — `limit_flags` are neither required nor permitted as fabrication.

- **Shock taxonomy**: `sentiment`, `liquidity`, `policy_keyword`, `fund_flow` (finance) plus `competitive_move`, `intel_event`, `public_event`, `outbreak`, `custom` (cross-domain). `shock_type` only selects the conditioning bucket and is covered by `params_hash`, so finance replays stay bit-identical.
- **DB constraint**: migration `v0.4.0__cross_domain_shock_types.sql` widens `chk_butterfly_shock_type` (DROP+ADD, idempotent); a regression guard pins the engine tuple and the CHECK together.
- **Pilot skills**: `sentiment-series-prepare` (event-driven data → equal-spaced grid + quality gate + evidence bundle), `sentiment-early-warning-watch` (domain action mapping: record / review+observe / escalate), `sentiment-fragility-report` (single-topic fragility report). The finance trim/veto mapping (R11) stays frozen and scoped to `butterfly-early-warning-watch`.
- **Fail-closed floors are domain-neutral**: below 30 same-type shock events the amplification leg refuses — sparse domains report the bucket as uncovered instead of padding the series.

---

## Reproducibility & audit

Every run is bound to the **reproducibility triple**: `(model_version, params_hash, random_seed)`.

- `params_hash` covers the full tunable set, the input profile (`kind`, `limit_flags`) and the seeded subsampling rule (`rule name` + `kept count`), so the same hash ⇒ the same numeric path (v3.1 L3).
- History subsampling above `max_history_points` is derived **deterministically from the run seed**.
- `replay_run(run_id)` rebuilds the exact payload from the stored `replay_input` + `request_params` snapshots and compares **every numeric field at tolerance 1e-9**; `params_hash_match` is checked.
- Input snapshots are stored for series ≤ 5000 points; larger series are explicitly marked `replayable = false`.
- **Failed/rejected runs are persisted too** (`success = FALSE`) — the audit trail shows rejections, not just successes.
- Idempotency: repeated submissions of the same `(task_id, run_type, params_hash)` return the existing run (`ON CONFLICT … DO NOTHING` + fallback select); failed runs stay retryable.
- Run + history rows commit in a single transaction (no half-written runs); row-level ownership scoping applies to all admin reads (owner + site, platform admins exempt).

---

## Early-warning levels & governance

| Level | Action | Delivery |
|-------|--------|----------|
| `info` | Record only | No action |
| `watch` | **trim** — suggest reducing exposure intervals | Risk suggestion; **no** automatic orders |
| `alert` | **escalate** to human + confirmation queue | Confirmed `alert` may suggest `veto`; **never auto-delivered to clients** |

- `alert` delivery is **frozen** until the M2 quantitative gate (backtest seven elements + independent reviewer sign-off) passes.
- Cooldown is **DB-authoritative** (default 6 h) so multiple workers cannot duplicate alerts.
- A trigger requires exceeding **both** the GARCH-filtered baseline envelope **and** the bootstrap p95 — missing either, no trigger.

## Dashboard statistics

| Metric | Type | Meaning |
|--------|------|---------|
| `butterfly_runs_total` | counter | Total analysis runs |
| `butterfly_high_sensitivity` | counter | Symbols with high sensitivity |
| `butterfly_active_warnings` | counter | Active warnings |
| `butterfly_compute_failures` | counter | Compute failures / rejections |
| `butterfly_placeholder_ratio` | gauge | Share of placeholder results |
| `butterfly_data_quality_blocked` | counter | Series blocked by the quality gate |
| `butterfly_warning_hit_rate` | gauge | Rolling warning hit rate (honestly `None` until M2 backtest) |
| `butterfly_level_flip_rate` | gauge | Warning-level flip rate (computed from the real event sequence) |

---

## Testing

```bash
python -m pytest plugins/butterfly_engine/tests -q          # 64 passed / 1 skipped (PG-less env)
python plugins/butterfly_engine/tools/mcp_server.py --list  # MCP self-test
python plugins/butterfly_engine/tests/perf/bench_p95.py     # P95 latency bench
```

Test suite coverage includes: golden tests (Logistic r=4, λ = ln 2 ≈ 0.693 quantitative baseline; Lorenz qualitative positive control; iid / random-walk / GARCH negative controls), idempotent replay, real-PG persist fidelity (`test_persist_realpg.py`), admin scoping/confirm semantics, i18n, gates, and performance.

## Performance (P95 latency, CI-measured)

| Series length | P95 | Notes |
|---------------|-----|-------|
| 600 points | ≈ 29.6 ms | Includes a 38 ms first-round warmup outlier; the other 6 samples run 3.7–10 ms |
| 3000 points | ≈ 16.4 ms | Independent re-measurement 2026-10-12 |

The 100 ms budget applies to the in-index window only; full-history scans are explicitly flagged via `perf_mode`.

---

## Documentation

| Doc | Path | Content |
|-----|------|---------|
| **Whitepaper** | [`docs/whitepaper.md`](docs/whitepaper.md) | Full method specification, validation, governance (English; Chinese version preserved at `docs/whitepaper_CN.md`) |
| **User guide** | [`docs/user-guide.md`](docs/user-guide.md) | Installation, configuration, MCP/API usage, admin workflow, result interpretation, troubleshooting |
| Model card — sensitivity | [`docs/model_card_sensitivity.md`](docs/model_card_sensitivity.md) | `butterfly.sensitivity` |
| Model card — amplification | [`docs/model_card_amplification.md`](docs/model_card_amplification.md) | `butterfly.amplification` |
| Model card — early warning | [`docs/model_card_early_warning.md`](docs/model_card_early_warning.md) | `butterfly.early_warning` |
| Model card — replay | [`docs/model_card_replay.md`](docs/model_card_replay.md) | `replay_run` |
| Audit report | archived outside the repo: `F:\Sites\butterfly_engine_reports_archive\audit_report_v0.1.2.md` | Third-party code audit findings C1–C12 / N1–N4 / F1–F13 |
| Changelog | [`CHANGELOG.md`](CHANGELOG.md) | Full version history 0.1.0 → 1.1.0 |
| Skills | [`skills/`](skills/) | Seven platform skills: butterfly-data-prepare, butterfly-fragility-report, butterfly-impact-assess, butterfly-early-warning-watch (finance) + sentiment-series-prepare, sentiment-early-warning-watch, sentiment-fragility-report (cross-domain pilot) |
| 中文说明 | [`README_CN.md`](README_CN.md) | 本文件的中文版 |

---

## Governance status

- Plugin version **1.1.0**, model version **0.1.0**; finance native, cross-domain pilot visible to finance/research/enterprise.
- `alert`-level delivery remains **frozen** pending M2 backtest (seven-element protocol §9.3) + independent reviewer sign-off.
- Response-drift weighting is currently an **unverified fixed term** (`drift_incremental_test = pending`); an M2 incremental predictive-power test will promote it (`passed`) or force its weight to 0 (`failed`) — the mechanism is already implemented in `engine.py`.

## License & ownership

Author: **VeroRun** · `plugins/butterfly_engine` · independent version line. See `plugin.json` for the full capability/permission manifest.
