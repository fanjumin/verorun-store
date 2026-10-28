# Chaos Engine

**Descriptive chaos / nonlinear-dynamics risk primitives — fail-closed · evidence-linked · replayable. No trading signals, no investment advice.**

Chaos Engine is a standalone VeroRun plugin that runs **phase-space reconstruction** and a family of **descriptive** nonlinear-dynamics measures on equally spaced time series: the Lyapunov exponent, Hurst exponent, correlation dimension, entropy, nonlinearity tests, a dynamics regime label, and a predictable-horizon upper bound. Every number only describes the dynamics *of the supplied window under historical conditions* — it is **not a forecast, not causation, not a rise/fall probability, and not a trading signal**; alerts always remain with butterfly_engine.

A **cross-domain pilot** (edition visibility: finance / research / enterprise) applies the same primitives to non-finance equal-spaced series — opinion, intelligence and competitive series — through the new domain profiles and the `sentiment-dynamics` Skill. Cross-domain runs must pass `input_kind="state"` explicitly (the engine default `price` would trigger a return conversion).

> Versions: model contract **1.1.1** / plugin package **1.2.2** (`min_app_version 0.63.0`). The 1.2.0 package release is the cross-domain pilot (domain profiles, edition visibility, first Skill) with **no numeric-path change** — the model contract stays 1.1.1. Version discipline: [CHANGELOG.md](CHANGELOG.md).
> 中文完整文档：[README_CN.md](README_CN.md)。

---

## 1. Capabilities (9 MCP tools)

| Capability | MCP tool | What it measures | Evidence required |
|---|---|---|---|
| `chaos.reconstruct` | `chaos_reconstruct` | AMI delay τ + Kennel FNN embedding dimension m (P0 shared prerequisite) | evidence bundle by default |
| `chaos.lyapunov` | `chaos_lyapunov` | Rosenstein largest Lyapunov exponent λ, doubling time t_d, IID surrogate veto | — |
| `chaos.hurst` | `chaos_hurst` | R/S + DFA Hurst exponent H with moving-block bootstrap 95% CI | — |
| `chaos.fractal_dim` | `chaos_fractal_dim` | Grassberger–Procaccia correlation dimension D2 | — |
| `chaos.entropy` | `chaos_entropy` | Permutation entropy + sample entropy + surrogate-null z score | — |
| `chaos.nonlinear_test` | `chaos_nonlinear_test` | BDS + 0-1 test + AAFT, three-test joint verdict | — |
| `chaos.regime` | `chaos_regime` | Descriptive regime label trending/mean_reverting/chaotic/noisy | — |
| `chaos.horizon` | `chaos_horizon` | Predictable-window upper bound t_d = ln2/λ (six gates incl. IID twin gate) | — |
| `chaos.evidence_link` | `chaos_link_evidence` | Self-contained evidence-bundle association (PII scan) | — |

Transport: stdio JSON-RPC 2.0 MCP server (`timeout 30s`); sole runtime dependency: `numpy`.

---

## 2. Boundary statement (product contract, never bypass)

1. **Descriptive measure, not a prediction**: results characterize statistics/dynamics of the window that already happened.
2. **Association, not causation**: conditional/historical association only.
3. **No trading signals, no investment advice**: never emits buy/sell calls, target prices, rise probabilities, or alerts (alerts belong to butterfly_engine).
4. **Stochastic inputs are never endorsed**: iid noise and strictly regular/periodic series cannot obtain a "chaos / positive predictability" verdict on any indicator (enforced by surrogate-null gates).
5. **Fail-closed**: every input/persistence failure returns an explicit error code; there is never a silent downgrade to the memory store.
6. **Replayable**: every successful run carries the reproducibility trio `model_version + params_hash + random_seed`.
7. **No action advice (cross-domain)**: on non-finance series the rendered boundary is the domain-neutral statement "descriptive measure, does not constitute any action advice" (`boundary.no_action_advice`); the finance wording (`no_investment_advice`) is unchanged for finance runs.

---

## 3. Quick start

### Requirements & install

- VeroRun app ≥ 0.63.0; Python ≥ 3.11; `numpy` (pure numpy, no scipy).
- Edition visibility: `compatible_editions: ["finance", "research", "enterprise"]` (finance-only before 1.2.0).
- Install via the kernel plugin manager; PF-10 migrations run automatically and create **4 tables** under the PostgreSQL `chaos_engine` schema (a memory store is available for tests/standalone use without PG).
- Enablement prerequisite (release registration): the plugin must first be claimed in an agent-role catalog, included in an edition manifest, and refreshed in the integrity manifest before it is reachable in that edition; see "Governance & release status".

### MCP self-check

```bash
python plugins/chaos_engine/tools/mcp_server.py --list     # tool list goes to stderr (stdout is permanently reserved for protocol frames)
```

### Minimal JSON-RPC call (stdio)

```jsonc
// -> send to stdin, one frame per line
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"chaos_reconstruct","arguments":{
  "symbol":"000001.SZ",
  "values":[/* equally spaced numeric series; converted to simple returns internally in price mode */],
  "input_kind":"price",
  "asset_type":"stock",
  "window":252,
  "random_seed":20240101,
  "evidence_bundle_id":"ev-2026-0001",
  "task_id":"task-abc"
}}}
```

Fixed engine envelope:

```json
{ "success": true, "data": { }, "trace": { }, "error": null, "model_version": "1.1.1" }
```

`trace` is mandatory audit information (method_chain, subsample, budget, persistence, compute_ms, ...).

### Run the tests

```bash
uv run --with numpy --with pytest --with pyyaml python -m pytest plugins/chaos_engine/tests -q
```

---

## 4. How it works (one page)

```
raw series
  |  quality_gate (length/DoS -> return conversion -> non-finite -> >20% flat -> near-zero variance)
  |  optional profile: stock(252) / futures(500) / crypto(720) / sentiment(168) / intel(120) / competitive(60) / research(252) / generic(60)
  v
window (last N points) -- 20000-point contiguous-tail ceiling
  v
AMI first significant minimum tau -- Kennel FNN convergence dim m (search budget: 3000-point contiguous tail)
  v
+----------------+----------------+----------------+----------------+
 lyapunov         hurst            fractal D2       entropy
 +IID surr gate   +bootstrap CI                     +surrogate null
 +----------------+----------------+----------------+----------------+
 nonlinear_test (BDS + 0-1 + AAFT)                    horizon (six gates)
        +------------------> regime synthesis <----------------------+
```

Deterministic, pure numpy, no network, no global random state; all randomness derives from the caller `random_seed`.

---

## 5. Input contract

| Field | Description |
|---|---|
| `symbol` | Required, regex `[A-Za-z0-9._-]{1,64}` |
| `values` | Required, equally spaced numeric array (raw prices / returns / state variable) |
| `input_kind` | `price` (converted internally to **simple returns**; a zero previous price contributes 0) / `return` or `state` (used as-is) |
| `asset_type` | `stock` / `futures` / `crypto` (finance) plus `sentiment` / `intel` / `competitive` / `research` / `generic` (cross-domain pilot, 1.2.0) — window and AMI-lag presets; unknown values fall back to defaults |
| `window` | Last N points; integer in [2, max_input_points] |
| `random_seed` | Integer [0, 2^31-1]; **0 is a legal seed**; drives bootstrap/surrogates |
| `tau` / `m` | Optional embedding overrides (lyapunov tool); AMI/FNN auto-decide when omitted |
| `evidence_bundle_id` | Evidence bundle id; required for reconstruct when `require_evidence=true` |
| `task_id` | Idempotency key; repeated deliveries with the same `(task_id, run_type, params_hash)` reuse one run |
| `triggered_by` / `owner_user_id` | Trigger source / owning user (audit) |

Fixed quality-gate order (first failure wins): raw length -> post-conversion minimum length -> non-finite -> unchanged-neighbor ratio >20% -> variance <1e-18. The last two can be skipped with `require_data_quality=false`; the first three are always enforced.

### Error codes

| Code | HTTP | Meaning | Persisted failure row |
|---|---|---|---|
| INVALID_SYMBOL | 400 | Illegal symbol format | ✔ |
| INSUFFICIENT_DATA | 400 | Shorter than minimum after conversion | ✔ |
| NON_FINITE | 400 | Contains NaN/inf | ✔ |
| QUALITY_GATE_FAILED | 422 | >20% unchanged neighbors (halt/flat/illiquid) | ✔ |
| DEGENERATE_SERIES | 422 | Near-zero variance | ✔ |
| EVIDENCE_REQUIRED | 403 | Missing evidence bundle | ✔ |
| INPUT_TOO_LARGE | 413 | Exceeds max_input_points (default 200k) | ✔ |
| INVALID_WINDOW | 400 | window non-integer or out of range | ✔ |
| INVALID_SEED | 400 | Seed not an integer in [0,2^31-1] | ✔ |
| COMPUTE_ERROR | 500 | Internal compute error (logged only, not persisted) | — |
| DB_UNAVAILABLE | 503 | Persistence failure, fail-closed | — |

---

## 6. Reproducibility, audit & persistence

- **Trio**: `model_version` (contract version) + `params_hash` (SHA-256 of canonical JSON parameter snapshot) + `random_seed`.
- **Hash-bound snapshot**: tau/m, fnn_threshold, ami_max_lag, window, n, seed, subsample, budget, and (lyapunov) surrogate-gate config; identical input + seed reproduces bit-for-bit.
- **4 tables** (PG schema `chaos_engine`): `chaos_runs` (19-column master, success and failure rows), `chaos_reconstruction_history`, `chaos_indicator_history`, `chaos_evidence_links`.
- **Failure rows**: 9 caller-attributable gate codes persist as `success=FALSE + error_code/error_message` (exceptions: COMPUTE_ERROR/DB_UNAVAILABLE).
- **Idempotency**: partial unique index `(task_id, run_type, params_hash) WHERE task_id IS NOT NULL`; replays return the same run_id.
- **PII**: evidence snapshots are scanned for phone/mobile/email/id_card keys and masked before storage (`pii_guard`).

---

## 7. Compute budgets (G3, inside the 30s MCP timeout)

Two deterministic layers, both using a **contiguous tail** (same semantics as the "last N points" window; random thinning breaks the Takens equal-spacing premise and changes FNN convergence):

| Stage | Cap | Audit field |
|---|---|---|
| Memory ceiling | 20,000 points | `subsample: {rule:"contiguous_tail", kept, total}` |
| Embedding search (AMI/FNN) | 3,000 points | `budget.search` |
| fractal D2 | 6,000 points | `budget.metric` |
| lyapunov | 2,000 points (1 observed + 19 surrogates) | `budget.metric` |
| horizon | 2,000 points (1 observed + 19 surrogates) | `budget.metric` |
| hurst / entropy / nonlinearity | self-truncate internally | — |

Every triggered budget enters params_hash; measured worst-case runs at 20,000 legitimate points complete within the 28s test threshold.

---

## 8. Configuration

| Key | Default | Description |
|---|---|---|
| `default_window` | 60 | Last-N window when a request specifies none |
| `min_data_points` | 60 | Fail-closed floor after return conversion |
| `max_input_points` | 200000 | Hard DoS upper bound on raw series length |
| `embedding.max_dim` | 10 | Candidate embedding-dimension ceiling [2..10] |
| `embedding.fnn_threshold` | 0.10 | FNN convergence ratio |
| `embedding.ami_max_lag` | 50 (crypto 40) | AMI first-minimum search ceiling |
| `embedding.max_embed_points` | 20000 | Contiguous-tail ceiling |
| `require_evidence` | true | reconstruct without a bundle fails EVIDENCE_REQUIRED |
| `require_data_quality` | true | Toggles flat-segment/zero-variance checks only (other gates always on) |
| `pii_guard` | true | PII scan on evidence snapshots |
| `retention_days` | 730 | Run-history retention |

---

## 9. Testing & validation

134 pytest cases pass: gates/envelope, P1 numerical goldens (Logistic/Hénon/sine/white-noise controls), PgStore 19-column SQL contract, real-subprocess MCP smoke, vectorization bit-equality, budget time bounds, the N2 noise false-evidence gate, the N3 ceiling golden, cross-domain asset-profile table contract and the i18n boundary-key guard. Methods, goldens and limitations:

| Document | Link |
|---|---|
| Technical whitepaper (English primary) | [docs/whitepaper.md](docs/whitepaper.md) |
| 技术白皮书（中文） | [docs/whitepaper_CN.md](docs/whitepaper_CN.md) |
| User guide (English primary) | [docs/user-guide.md](docs/user-guide.md) |
| 用户使用说明（中文） | [docs/user-guide_CN.md](docs/user-guide_CN.md) |
| Per-primitive model cards (Chinese, 8 cards) | [docs/](docs/) `model_card_*.md` |
| Changelog | [CHANGELOG.md](CHANGELOG.md) |

---

## 10. Governance & release status (stated honestly)

- Code-level fixes are closed and verified by 134 local cases plus real PostgreSQL round-trip re-audit; model contract **1.1.1** (unchanged by the 1.2.0 cross-domain pilot — no numeric-path change).
- Reachability in a production edition is gated by agent-role catalog claim + edition include + integrity-manifest refresh; before that registration completes the plugin is unreachable in that edition (a release gate, not a code defect — item G1).
- Licensing and integrity are governed by VeroGuard / EULA; this plugin contains no bypass mechanism.
