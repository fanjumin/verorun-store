# Bayes Engine

> Version 1.0.1 — first stable release (2026-10-11).

Platform-level generic Bayesian inference for VeroRun: prior -> likelihood ->
posterior -> posterior predictive, with evidence-linked, replayable and
auditable probabilities. Business semantics live in Skills; the plugin makes
no LLM calls and has no network egress.

## Features

- Four model families: Beta-Binomial, Dirichlet-Multinomial, Normal-Normal,
  discrete hypothesis spaces;
- Exact quantiles and credible intervals (pure standard library, no
  numpy/scipy);
- Family-wide per-observation weighting (tempered posterior), Normal-Normal
  included;
- Recoverable Bayes factors and log marginal likelihood (discrete: exact;
  conjugate families: closed-form cumulative at weight = 1);
- Posterior predictive distributions and likelihood robustness bands;
- PII-gated (recursively masked) evidence snapshots; full run history with
  replay;
- Capability API with channel-based ownership: the SSO channel is pinned to the
  session user (cross-user access and owner spoofing rejected with 403), while
  the internal-token channel may set the owner; an optional strict evidence mode
  requires a bundle id.

## Domain Skills

| Skill | Domain |
|---|---|
| belief-update | generic belief updates |
| source-reliability | finance source reliability |
| signal-calibration | finance signal calibration |
| sentiment-credibility | public opinion / sentiment |
| research-belief-revision | research hypothesis revision |

## Getting started

See [USAGE.md](USAGE.md); capability specification in
[docs/whitepaper.md](docs/whitepaper.md).

## Tests

```
python tests/test_distributions.py
python tests/test_engine.py
python tests/test_routes.py
python tests/test_mcp.py
python tests/test_pg_integration.py
```

## Kernel contract dependencies

Pure math (`distributions.py` / `special.py`) is stdlib-only. The platform
integration layer depends on host-shared kernel contracts (standard v1.8):

- `plugins._base.db` — pooled connections / borrow-and-return discipline (§9.1);
- `plugins._base.pii` — PII guard (stdlib fallback shipped);
- `plugin_manager.{base,event_bus,logger}` — lifecycle, event bus, logging;
- `routes.admin._require_admin` — admin SSO guard for `/admin/bayes`;
- `INTERNAL_SERVICE_TOKEN` / `X-Internal-Token` — capability API auth (§11.1).

These are not plugin.json `dependencies` (host-shared modules, §9.1).

## Watermark

No watermark is carried in source. At publish time the platform's
`tools/publish_plugin.py` injects a signed build manifest and canary markers
into a temporary copy; runtime output carries response provenance
(standard §17).
