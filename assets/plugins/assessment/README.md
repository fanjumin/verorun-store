# Psychometric Assessment (`assessment` v1.1.0)

Versioned instruments, CTT scoring with explicit point maps, norm conversion,
reliability estimation and an auditable answer trail. The plugin owns its
schema, routes and pages; the generic questionnaire parts come from a shared
kernel.

- **Identifier:** `assessment`
- **Version:** 1.0.0
- **Category:** ai_agent · **Menu:** Analysis → Instruments / Sessions / Results
- **Min app version:** 0.10.0 · **Core API:** 1
- **Editions:** enterprise, standard, pro, minipro, finance
- **Agent role:** `survey` (reused; no new role was introduced)
- **Permissions:** `routes`, `scheduler`, `admin:access`

## Dependencies and boundaries

```
survey ─┐
        ├──> plugins._survey_core       (no imports between the two plugins)
assessment ─┘
```

`assessment` depends on the questionnaire kernel `plugins/_survey_core` and on
nothing else; neither plugin imports the other.

- `_survey_core` has no `plugin.json`, and its `_` prefix is skipped by
  `plugin_manager/discovery.py`, so it never appears in the plugin centre.
- Kernel version is negotiated at enable time: the plugin declares
  `core_api: 1`, and a mismatch refuses to enable.
- `assessment` owns the PostgreSQL schema of the same name, with no foreign key
  pointing into `survey`. Uninstall runs `DROP SCHEMA assessment CASCADE` and
  leaves survey data untouched.
- Answering is login-only and results are per-subject; the plugin renders no
  respondent UI, so an external frontend or mini program owns presentation.

## Feature surface

**Instruments as a three-level model.** `instrument → version → item`. Drafts
are editable; publishing stamps `frozen_at` and a database trigger rejects any
later change to items, `scoring_json`, `algorithm_version` or `version`, and
rejects unfreezing. Editing a published instrument means cloning a new version.

**Scoring.** CTT with a single hard rule: the value of an answer comes **only**
from an explicit `point_map` on the item. No derivation, no implicit identity
map. Rules:

- A missing `point_map` key is an error, not a silent zero.
- `reverse: true` requires a stored mirrored map (`mirror_point_map`), so the
  inversion is auditable in the frozen version instead of being recomputed at
  runtime.
- Missing answers follow one of three declared strategies — `reject`,
  `mean_impute`, `prorate` — and every imputation is recorded in `imputed` and
  shown in the report.
- Aggregation per dimension is `sum` or `mean`.
- `range` is display-only metadata and never participates in scoring: a Likert
  response configured as `a1=4, a2=5` scores exactly 9, because both points come
  from the maps.

**Norm conversion.** Three-tier lookup: exact stratifier → total norm for that
dimension → no norm. Percentile is linearly interpolated between adjacent raw
scores, never extrapolated. From percentile the service derives z and T. When
no norm applies, `graded` is `False`, `band` is `None` and the dimension is
pushed into `ungraded` — the UI must render it as "ungraded", never drop it.

**Reliability.** Cronbach's alpha and split-half per dimension plus a total,
computed from the submitted item matrix. Below `reliability_min_sample`
(default 30) the result carries `low_sample_warning`; alpha never participates
in an individual's grading.

**Answer quality.** Four flags computed at submit time: `too_fast` (below
`min_seconds_per_item`), `straightlining` (a run of identical responses),
`inconsistent_pairs` (declared item pairs whose directions disagree) and
`social_desirability_high` (a declared dimension above its threshold). Flags
are shown **inside the report**, not merely in an operations dashboard.

**Sessions.** The answering path is constrained as follows: login is mandatory
(no anonymous channel, no share token), the consent text hash is stored and a
missing consent is rejected, a retest interval set on the batch
(`retest_min_days`, default `0` = off) blocks re-administering the same
instrument to the same subject too soon, and scoring runs before anything is
written — a scoring failure rejects the submission outright, so no half-written
result can exist.

**Audit and erasure.** Item-level responses and every individual report viewed
by an administrator require `?confirm=1` and write an audit row. The audit
`action` column is constrained to a whitelist: `view_detail`, `export`,
`disclose_result`, `delete_subject`, `void_session`, `publish`, `rescore`.
Erasure nulls `subject_user_id` (the partial unique index then frees the
subject to be assessed again) while keeping aggregate scores for group
statistics, and records the action.

## Blueprints and endpoints

| Blueprint | Prefix | Auth | Scope |
|---|---|---|---|
| `admin_bp` | `/plugin/assessment` | admin session | instruments, versions, publish, norms, batches, sessions, results, audit, erasure |
| `answer_bp` | `/assessment` | login required | `GET /answer/<id>` (answering page), `POST /api/start`, `POST /api/<sid>/submit`, `GET /api/<sid>/report`, `GET /api/<sid>/report/pdf` |
| `report_bp` | `/assessment/report` | page only | `GET /<sid>` renders the printable report |

Admin API surface: `GET|POST /api/instruments`, `GET /api/instruments/<id>`,
`PUT .../versions/<vid>`, `POST .../versions/<vid>/clone`,
`POST .../versions/<vid>/publish`, `GET .../reliability`, `GET|PUT .../norms`,
`GET|POST /api/batches`, `GET /api/sessions`, `GET /api/sessions/<id>`,
`GET /api/sessions/<id>/report`, `POST /api/sessions/<id>/rescore`,
`POST /api/sessions/<id>/void`, `GET /api/audit`,
`DELETE /api/subjects/<subject_user_id>`, `GET /api/error-codes`.

The answering page (`GET /assessment/answer/<id>`) is self-contained: it consumes
the existing `start` / `submit` endpoints only, renders the whitelisted question
projection (`point_map` / `reverse` / `dimension` never reach the browser), and
dispatches `change` on every custom control. Reports download as PDF through
`GET /assessment/api/<sid>/report/pdf` — drawn server-side with reportlab (no
Chrome on target machines; `STSong-Light` CID font for CJK), permissioned by the
same gate as the web report (respondent directly, admin with `?confirm=1` +
audit). `rescore` re-runs scoring on an existing result without touching session
state, and is audited.

## Scoring configuration

`scoring_json` lives on the version, not on the instrument, so a published
version keeps the rules it was scored with.

```json
{
  "dimensions": [
    {"key": "d_extra", "label": "Extraversion", "items": ["a1", "a2"], "aggregate": "sum"}
  ],
  "missing": {"strategy": "prorate", "min_answered_ratio": 0.8},
  "report": {
    "type": "threshold",
    "total_dimension": "d_total",
    "thresholds": [
      {"dimension": "d_extra",
       "bands": [
         {"min": 0, "max": 6, "level": "L", "label_key": "norm.low"},
         {"min": 7, "max": 10, "level": "H", "label_key": "norm.high"}
       ]}
    ]
  },
  "quality": {"min_seconds_per_item": 1.5, "straightline_run": 6,
              "inconsistent_pairs": [["a1", "a7"]]}
}
```

Validation rejects overlapping bands, unknown item or dimension references,
duplicate dimension keys, nested rule combinators, and a `total_dimension` that
collides with a declared dimension (a synthesised key such as `d_total` is
allowed; shadowing a real dimension is not). Items referenced by no dimension
produce a `unused_items` warning rather than a hard error — the respondent
answered them but they score nothing, which is usually a configuration
mistake and must be visible in the designer.

`report.type` is either `threshold` (bands per dimension) or `typology` (ordered
rules, first match wins, with `unclassified` as the fallback). Explanation text
is referenced by i18n key only: putting prose into `scoring_json` would make a
frozen version impossible to re-word and would import un-auditable natural
language into the scoring config.

## Report structure

Fixed six blocks in reading order: `summary` (total, band, one-line
conclusion), `profile` (raw / percentile / T per dimension), `interpretations`
(i18n keys), `quality` (only when a flag exists), `disclaimer` (fixed key,
cannot be disabled) and `meta` (instrument version, norm version, algorithm
version, administered at). A result missing any of the three version
identifiers is marked non-reproducible.

## Database

Ten tables in schema `assessment`, created by `migrations/v1.0.0_init.sql`
through `BasePlugin.apply_sql_migrations` (ledger + advisory lock + one
transaction per file):

`assessment_instruments`, `assessment_instrument_versions`, `assessment_items`,
`assessment_norms`, `assessment_batches`, `assessment_sessions`,
`assessment_results`, `assessment_session_items`, `assessment_rate_events`,
`assessment_audit_log`.

Two triggers guard immutability: `trg_asmt_items_frozen` and
`trg_asmt_versions_frozen`. Both fire on `INSERT OR UPDATE` only — deliberately
not on `DELETE`, because blocking deletes would make the plugin's own uninstall
cascade collide with its guard; deletion is controlled by foreign keys instead
(a version that has results cannot be removed).

## Configuration

Plugin-level keys (written to the registry, injected once at instantiation, so a
change requires a service restart or plugin re-enable):

Only **one** of the six declared keys currently has a reader. The other five are
declared in `config` / `settings_schema` and injected, but no code path reads
them — changing them has no effect. Do not treat them as tunable until they are
wired.

| Key | Default | Read by |
|---|---|---|
| `session_ttl_days` | 90 | `scheduler.run_sweeper` — **in effect** |
| `max_items_per_instrument` | 300 | nothing; `instruments` uses its own `DEFAULT_MAX_ITEMS = 300` |
| `reliability_min_sample` | 30 | nothing; `instrument_alpha` keeps its own default argument |
| `min_seconds_per_item` | 1.5 | nothing at plugin level; quality reads `scoring_json.quality.min_seconds_per_item` per version |
| `stats_snapshot_minutes` | 15 | nothing; `dashboard_metrics()` always queries live |
| `sweeper_batch_size` | 500 | nothing; the sweep is a single `UPDATE` (also absent from `settings_schema`) |

The sweeper runs daily at 03:45 and is idempotent. It only voids
`in_progress` sessions past the TTL and purges rate events; **submitted
sessions and their results are never expired**, because a report must remain
reproducible. The plugin registers the job twice under two processes and is
safe there by design.

## i18n

`i18n/en.yml` and `i18n/zh-CN.yml`, 132 keys each, key sets identical. Seeded via
`seed_plugin_translations("assessment", I18N_DIR)` at install time; when the
directory is absent the step is skipped silently. Metadata keys (`plugin.*`, `menu.*`) carry readable English;
user-visible copy is an English source string mapped to Chinese. Page templates
inject `window.__t` and resolve runtime copy through a single `t()` helper,
matching the survey plugin's frontend pattern. Server-drawn PDF copy goes through
`i18n._()` with `resolve_locale()` (`?lang=` → cookie → `Accept-Language`);
placeholders such as `Page {page}` keep CJK word order intact.

Template names must be globally unique: the survey plugin also ships a result
page, so this plugin's report template is `assessment_report.html`. Flask's
Jinja loader resolves across plugins by registration order, and a shared name
silently renders the wrong template.

## Tests

Pure-function tests, no database and no Flask needed:

```bash
python -m unittest discover -s plugins/assessment/tests -p 'test_*.py' -v
```

100 cases cover point maps and reverse maps, the three missing strategies,
dimension aggregation, band and typology validation, norm interpolation and
the three-tier fallback, alpha and split-half, the four quality flags, report
assembly including the visible `ungraded` guarantee, the answering-page render
whitelist, PDF structure plus the five compliance red lines in both locales,
norm row validation, unique-violation detection, and admin form wiring.
HTTP and SQL behaviour was verified on a live deployment: end-to-end 63
assertions (instrument → publish → freeze rejection → answering page → start →
submit → report → PDF → norm import → rescore → percentile → out-of-range
stays ungraded → quality flags → batch retest interval → admin confirmations
→ audit → erasure → bilingual PDF), plus 14 browser-level assertions covering
the norm-import and batch-creation forms (including client-side blocking of
missing `source` and out-of-range percentiles).

## Known gaps

- **No visual item designer.** Instruments are created by `POST
  /api/instruments` with a full `questions` + `scoring` payload. The admin page
  can list, inspect, publish, clone and manage norms, but its "create" button
  submits an empty payload and the API rejects it with `scoring_dims_empty`;
  the page is an operations shell, not an editor.
- **`hooks.provides` declares `assessment.session.submitted`, but nothing
  emits it yet.** The declaration is ahead of the wiring.
- **No MCP tools.** Item-level responses and results are deliberately not
  exposed to agents in bulk.
