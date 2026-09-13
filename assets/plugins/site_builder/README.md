# AI Site Builder (site_builder)

> LLM-driven website builder for VeroRun: prompt templates, site tasks, unified design tokens, style presets, page tree & themed rendering gateway.
> Category: `content` · Agent role: `builder` · Version: `2.8.0` (plugin.json) · Min app version: `0.10.0` · Author: VeroRun

## Overview

AI Site Builder turns a natural-language requirement into a complete multi-page website: it parses the request, plans the site structure, and generates brand identity, theme, navigation, page sections and legal documents through the system Master Agent (Athena). Everything is written to a **draft area first**; a publish action then promotes it to production, with a version snapshot taken before every publish so any state can be rolled back.

The plugin is deliberately **decoupled from the core**:

- All plugin-owned data lives in a dedicated PostgreSQL schema `site_builder` inside the main `appdb`, accessed through the shared connection pool (no separate database server).
- Core/shared data (`cms_blocks`, `cms_posts`, brand settings) is **not** read directly: it is reached through the main site's internal API (`/api/internal/*`) via `internal_client.py`, with LRU cache + fallbacks for reads and real-time pass-through for writes.
- The plugin registers **5 blueprints**: `/admin/site-builder/*` (build console), `/admin/site-settings/*` (design tokens), `/shop/*` (user-facing plugin store), `/page/*` and `/site/*` (public rendering gateway).

## Highlights

| Area | What it gives you |
|------|-------------------|
| Generation pipeline | Parse requirement → plan preview (parallel LLM calls) → execute build DAG → draft |
| Industry templates | 6 built-in YAML prompt templates (corporate / ecommerce / education / iot_platform / local_service / personal) |
| Design tokens | Unified token system (brand/colors/typography/nav/footer/spacing/radius/shadows/glass/gradients/motion/layout/seo) |
| Style presets | 6 mainstream design styles applied with one click or auto-detected from the prompt |
| Plugin capability linking | Auto-detects capabilities declared by other enabled plugins (`ecommerce` → products/cart blocks + cart nav entry, `iot` → device_status) |
| Control blocks (Phase 1) | 19 rich block types (media_video/media_audio, gallery, carousel, chart, table, price_table, code_block, map, form_contact, countdown, counter, fx_hero, faq, tabs, timeline, social_links, embed, reviews) + ad_slot; schema-sanitized structured data, no free-form HTML |
| Public lead capture | `form_contact` posts to `POST /page/api/contact` → `sb_contact_leads` inbox (validation, rate-limit, honeypot) |
| Page tree (M1) | Hierarchical multi-page site model `sb_pages`, materialized paths, 3-level depth, soft delete, 301 redirects on move |
| Themed rendering | `rendering/` kernel: active-theme registry, template hierarchy (`page-{slug}` → `page-{type}` → `page` → `index`), per-block partials, block render hooks |
| Draft / publish / rollback | Everything edits a draft; publish snapshots tokens + blocks + documents + page tree into `site_versions` (max 30), restore is one click |
| Preview-as-Editor | Desktop H5 preview with in-place editing: inline text edit, drag sort/add, color & spacing panels, nav editor, image upload, undo/redo |
| Public gateway | `/page/<path>` published rendering, dynamic `/site/sitemap.xml`, `/site/robots.txt`, branded 404 |
| i18n | Full bilingual (en / zh-CN) admin UI — 347 keys × 2 in plugin `i18n/` plus 94 × 2 in `site_settings/i18n/` |

## How the build pipeline works

1. **Parse** — `SiteBuilderEngine.parse_requirement()` asks the LLM to extract structured fields (brand name, tagline, audience, style…) from the raw user input; falls back to template defaults on failure.
2. **Plan preview** — `generate_plan()` fills the template's per-field prompts and fires brand / navigation / footer / page / document LLM calls in parallel (`ThreadPoolExecutor`, up to 8 workers). Output is **hard-validated**: page sections must be a non-empty list of whitelisted `block_type`s and nav must be a non-empty `nav_items`; invalid JSON is retried once with a higher `max_tokens` budget and a stricter instruction, then degrades gracefully (empty sections / empty nav) instead of failing the build. JSON parsing retries 4000 → 6000 tokens to survive truncation.
3. **Execute (DAG)** — `execute_plan()` applies the plan in dependency order, always to the **draft** area:
   `brand settings → theme (depends on brand colors) → navigation + footer + footer articles → page blocks (per page) → legal documents`, recording per-step results into a task record (`site_builder_tasks`, id `SB-YYYYMMDD-XXXXXXXX`).
4. **Publish** — `POST /admin/site-builder/publish` archives a `pre_publish_*` snapshot + a numbered version (`v1`, `v2`, …), promotes `draft_json → token_json`, publishes blocks/documents by scope (pages + slugs derived from the draft), and marks the involved `sb_pages` as `published`.

Minimal modification (`POST /admin/site-builder/modify`) supports incremental edits: the LLM receives a page summary and returns a delta (`modify_block` / `add_block` / `delete_block` / `reorder`). Blocks that are already published are honestly rejected (`edit a draft version instead`) — no silent no-ops.

## LLM integration

- Uses the system-preset **Athena** master agent (`agent_matrix`); the plugin never creates its own agents (plugin standard §2.2).
- Calls go through `agent_matrix.engine.UnifiedLLM` with `module='site_builder'`; cached agent/engine instances.
- The 6 YAML templates in `prompts/` define per-industry defaults, page lists, document lists and full prompt text per page/document (keywords `{品牌名称}` `{行业}` … are filled by the engine). Templates marked `capabilities_required` only appear when the matching capability is enabled (e.g. `ecommerce` needs the shop plugin). Legacy overly-specific templates (`tech_company`, `restaurant`, `law_firm`) are auto-disabled on seed (data kept, `is_active=0`).

## Style presets

Six presets are defined in code (`style_presets.py`); each is a complete visual token set applied only over visual sections (brand/nav/footer/seo are preserved, idempotent):

| Preset | Trigger keywords (auto) | Character |
|--------|-------------------------|-----------|
| `modern_minimal` 现代极简 | minimal / modern / clean / 极简 / 现代 / 简洁 | whitespace, mono-accent, thin borders |
| `glassmorphism` 玻璃拟态 | glass / frosted / 毛玻璃 / 玻璃拟态 | translucent glass cards, gradient background |
| `dark_tech` 暗黑科技 | dark / tech / cyber / 暗黑 / 科技 | deep dark, neon gradients, glow |
| `editorial` 杂志风 | magazine / 杂志 / 报纸 | serif headings, typographic contrast |
| `neumorphism` 新拟态 | neumorph / soft ui / 新拟态 | soft dual shadows, low saturation |
| `duotone` 双色调 | duotone / 双色调 | two-color gradient overlays, high contrast |

Consumption chain: **admin picker** (draft tokens) / **AI auto-detection** (`resolve_style_preset()` on raw user text first, then on LLM-normalized style, then template default) / **rendering** (CSS variables). Preset extension sections (`glass` / `gradients` / `motion` / `layout`) are optional everywhere — old data without them falls back through CSS `var()` defaults with zero regression.

## Plugin capability linking

`capabilities.py` scans the enabled plugins and aggregates the capabilities they declare in their own `plugin.json` (`site_capabilities`) — no plugin name is hard-coded. Degradation rule: if the plugin manager is unavailable / query fails / no declarations → empty set, behavior identical to older versions.

| Capability | Extra block types | Render-side injection |
|-----------|-------------------|----------------------|
| `ecommerce` | `products`, `cart` | Cart entry appended to preview & published nav (idempotent, URL already present → skip) |
| `iot` | `device_status` | — |

Templates declare `defaults.capabilities_required`; the template list endpoint filters them by the current capability set, and `generate_plan()` passes the extra block types into the section validator.

## Control blocks & front-end runtime (2026-09, Phase 1)

On top of the 10 classic block types, a **control layer** of rich blocks was added. Controls are fully schema-driven: the LLM only emits structured JSON that is field-level sanitized before storage, and rendering goes through trusted partials + `sb-blocks.js` — no free-form HTML from the model ever reaches the browser.

### Block types

`block_schemas.py` defines two sets:

| Set | Types | Availability |
|-----|-------|--------------|
| `BASE_CONTROL_BLOCK_TYPES` (19) | `media_video`, `media_audio`, `gallery` (lightbox), `carousel`, `chart` (ECharts on demand), `table`, `price_table`, `code_block`, `map`, `form_contact`, `countdown`, `counter`, `fx_hero` (gradient/glow/typewriter), `faq`, `tabs`, `timeline`, `social_links`, `embed` (trusted-host allow-list), `reviews` | Always available (capability-independent) |
| `ad_slot` | Advertising container; schema allows `position` enum only, no free-text injection | Rendered empty unless an ad is served |

### Structured data pipeline

- `PageGenerator` writes control blocks **one block per DB row**: the LLM payload goes entirely into `extra_json` (sanitized) and `content` is left empty — free-form HTML from the model is prohibited (XSS convergence).
- `sanitize_sections()` ([block_schemas.py](block_schemas.py)) runs field-level checks before the section validator: enum fallback, numeric clamping, URL tightening (`http(s)://` or same-site relative only — `//` and `javascript:` rejected), dangerous key removal (`on*`, `srcdoc`); sections that still fail are dropped. Both `generate_plan()` passes (first + retry) run it.
- `controls_catalog_text()` appends the "available control catalog + JSON example + rules" to each page prompt at runtime — a single source of truth that works for existing built-in templates without re-seeding.

### Front-end runtime

- `static/css/sb-blocks.css` (fully tokenized) + `static/js/sb-blocks.js` (lightbox, carousel, charts, code copy, FAQ/Tabs, animated counters, countdown, typewriter, embed, contact form, reviews — zero third-party dependencies, ECharts loaded on demand), injected via the built-in `layout.html` and the fallback `public/page.html`.
- Capability-gated global widgets: `commerce_enabled` injects the cart drawer (`cart-drawer.js`, `products` `source=mall` mode), `chat_enabled` injects the AI support bubble (`partials/chat_bubble.html`). When the provider plugin is disabled nothing is loaded.
- Cart nav URL corrected to `/mall/cart` (shop public prefix is `/mall`; the earlier `/shop/cart` pointed at the plugin-store page).

## Page tree & themed rendering gateway

### Page tree (M1, `pages/models.py`, schema `sb_pages`)

- Single source of truth for the site structure: `slug` mirrors `cms_blocks.page` (slug immutable after creation), `path` is a server-materialized path (`/a/b/c`, max 3 levels, cycle/depth/unknown-parent rejected by `build_paths()`).
- CRUD with soft delete (published pages are only taken offline; paths of hard-deleted rows are rewritten to `/deleted/<id>` to free the unique `(site_key, path)` index).
- Moving a published page appends the old path to `extra_json.redirect_from`; the gateway answers with a real **301**; self-referencing 301 loops are prevented.
- `rebuild_navigation()` regenerates `design_tokens.navigation.items` from the tree (top-level + children, URL `/page{path}`), writing both `token_json` and `draft_json`.
- Backfill (`backfill_pages()`, idempotent, `ON CONFLICT DO NOTHING`) syncs legacy published/draft pages into the tree at plugin init and on publish.

### Rendering kernel (`rendering/`, Unreleased)

| Module | Responsibility |
|--------|----------------|
| `theme_registry.py` | Active theme from `tokens.theme.active_theme_id`, default `builtin`; any error/unknown theme falls back to `builtin` (never 500) |
| `template_hierarchy.py` | Resolves `page-{slug}.html → page-{page_type}.html → page.html → index.html` |
| `block_names.py` | `block_type` → safe partial name; unknown types → `_fallback.html` (never raw output, never 500) |
| `block_registry.py` | Per-block render hooks: `register_block_partial(type, path)` / `register_block_renderer(type, 'pkg.mod.callable')`; hook exceptions fall back to the partial |

The public router `render_page` renders through `_render_themed()` (registry + hierarchy + partials); any template resolution error falls back to the legacy `templates/public/page.html` — zero 500. The built-in theme lives in `templates/themes/builtin/` (`layout.html` shell + `index.html` generic page + `blocks/*` partials for the 10 core types + `_fallback.html`); the 10 core block types render byte-identical to the legacy template (zero behavior regression).

### Public endpoints

- `GET /page/<path>` — published page only; branded 404 (reuses token nav/footer) otherwise
- `POST /page/api/contact` — public lead capture for `form_contact` blocks: validation, per-IP rate-limit, honeypot, stored into `sb_contact_leads`
- `GET /site/sitemap.xml` — dynamic sitemap from published pages (in-memory)
- `GET /site/robots.txt` — `Disallow: /admin/` + sitemap directive

## Design tokens & site settings (`site_settings/`)

`design_tokens` table holds two layers per `site_key` (`platform`): **`token_json` (production)** and **`draft_json` (work in progress)**. Editors always write the draft; explicit publish (`?publish=1` or the publish button) promotes it.

| Section | Content |
|---------|---------|
| `brand` | site_name, slogan, industry, brand_story, logo/favicon URL, company_name, contact_email |
| `colors` | primary/secondary/accent/background/surface/text_primary/text_secondary/border/error/success |
| `typography` | heading/body fonts, font scale, sizes, line height |
| `navigation` / `footer` | items with children; grouped links, legal articles, copyright, ICP & security numbers |
| `spacing` / `border_radius` / `shadows` | spacing scale, radii, shadow levels |
| `glass` / `gradients` / `motion` / `layout` | style-preset extension sections (optional) |
| `seo` / `meta` | title/description; generator & version info |

`token_renderer.py` emits CSS variables (`:root { --color-* / --font-* / --space-* … }`), nav HTML, footer HTML and brand meta — all values HTML-escaped (XSS-safe). `token_service.py` validates structure and exposes a JSON schema used to steer the LLM.

`site_versions` snapshots the full state (tokens + blocks + documents + page tree) on publish with auto-numbered labels (`v1`, `v2`, …; `pre_publish_*` snapshots skipped for numbering), keeps the newest 30, and `restore` replays the snapshot back into the **draft** (never auto-publishes; page-tree restore is safe-mode: fills gaps, never deletes extra pages).

## Admin console & editor

`templates/admin_site_builder.html` (inline partial, entry under System → AI Site Builder, `/admin/site-builder/prompts`) hosts four areas: AI generation flow, style-preset picker, page-tree management, and the site-settings editor.

`templates/ai_site_preview.html` + `static/` (served by `/admin/site-builder/preview-static/*`) is a desktop H5 **Preview-as-Editor** (mobile mini-program frame removed since v2.5.0):

| Frontend module | Role |
|-----------------|------|
| `api-client.js` | HTTP wrapper, auto-retry ×2, JWT from cookie |
| `state-manager.js` | undo/redo stack (20 steps), dirty flag, Ctrl+Z / Ctrl+Shift+Z |
| `inline-editor.js` | double-click inline text editing (`contenteditable=plaintext-only`, auto-save on blur) |
| `block-actions.js` | native HTML5 drag sort, drag-to-add, hide/show, delete |
| `nav-editor.js` | inline edit of navigation titles/URLs, add/delete items |
| `color-palette.js` | 10 color swatches + preset theme buttons |
| `spacing-slider.js` | sliders for section gap / card padding / body font size |
| `editor-toolbar.js` | floating toolbar: save, publish, panels, undo/redo |

Draft editor API (`/admin/site-builder/api/draft/*`): field update with whitelist (`title/subtitle/content/link_text/link_url/image_url/icon/extra_json`), batch reorder, soft delete (`extra_json.deleted`, also filtered at render time), insert block, token scope updates (deep-merge per scope), image upload (jpg/png/gif/webp only, ≤5 MB, stored under plugin `static/uploads/draft/`, SVG blocked).

## User-facing store (`/shop/*`)

P0-2 plugin store pages mounted on both main_site and admin: `/shop/` (plugins + skills tabs), `/shop/plugins/<identifier>`, `/shop/skills/<identifier>` (with one-click install). Data is fetched in-page from the public plugin_manager endpoints; purchase/subscription is intentionally out of scope (prices displayed only).

## Data model (schema `site_builder`)

| Table | Purpose |
|-------|---------|
| `site_builder_prompts` | industry prompt templates (builtin + user-created; `is_builtin` protects deletion) |
| `site_builder_tasks` | build tasks: status, plan/result JSON, current step, error |
| `design_tokens` | production `token_json` + `draft_json` per `site_key`, version counter |
| `site_versions` | publish snapshots (tokens/blocks/documents/pages JSON), newest 30 kept |
| `sb_pages` | page tree: slug, parent_id, materialized `path`, page_type, layout, in_nav, status (`draft/published/unpublished/deleted`), seo_json, extra_json (`redirect_from`) |
| `sb_contact_leads` | public lead inbox from `form_contact` (`POST /page/api/contact`): site_key, page, name/email/phone/company/message, ip, created_at (indexed) |
| `schema_migrations` | applied migration files (run-once, advisory-lock serialized) |

Shared core tables (`cms_blocks`, `cms_posts`, brand) stay in the main DB and are only touched through the main site internal API endpoints: `/api/internal/brand`, `/api/internal/cms/draft-blocks`, `/cms/draft-documents`, `/cms/page-blocks`, `/cms/pages`, `/cms/draft-blocks/replace`, `/cms/blocks/update`, `/cms/blocks/order`, `/cms/blocks/delete`, `/cms/blocks/add`, `/cms/documents`, `/cms/publish` (auth via `X-Internal-Token`).

## Configuration

| Key | Default | Description |
|-----|---------|-------------|
| `MAIN_SITE_INTERNAL_URL` | `http://127.0.0.1:8083` | Main-site internal API base URL (`internal_base_url_env`) |
| `INTERNAL_SERVICE_TOKEN` | *(empty)* | Internal service auth token, must match main_site (`internal_token_env`) |
| schema | `site_builder` | Dedicated PostgreSQL schema in `appdb` (auto-created idempotently) |

Migration is executed on install/enable/setup with a transaction-level advisory lock (`pg_try_advisory_xact_lock`, key `SBMG`) so concurrent gunicorn workers never race DDL; `schema_migrations` records applied files; legacy `v2.1.0*` SQL files are archived and skipped (their tables are created by `init_tables()`). Uninstall keeps all data (schema preserved).

## Permissions & hooks

- Permissions: `admin:access`, `database:write`, `network:request`, `file:write`
- Hooks provided: `site_builder.build`, `site_builder.publish`, `site_settings.tokens`
- Admin API auth is self-contained: Bearer token or `sso_token` cookie → JWT validated → `is_admin` required; audit via structured plugin logger.

## API surface (route prefixes)

| Blueprint | Prefix | Purpose |
|-----------|--------|---------|
| `site_builder_bp` | `/admin/site-builder` | prompts CRUD, capabilities, style presets, preview/execute/publish, draft-data, preview-site, modify, tasks, page-summary, draft editor API, versions, page tree, static assets |
| `site_settings_bp` | `/admin/site-settings` | tokens GET/PUT, schema, CSS/render output, brand/navigation/footer/colors/typography sub-editors |
| `shop_bp` | `/shop` | plugin & skill store pages |
| `site_public_bp` | `/page` | published page rendering gateway |
| `site_public_site_bp` | `/site` | sitemap.xml, robots.txt |

Dashboard stats: `total_tasks`, `completed_tasks`, `total_prompts` (read from the plugin schema, idempotent).

## Getting started

1. The plugin ships in the default plugin directory — enable it in the admin plugin manager (installation auto-creates the schema, tables, seed templates, and backfills existing pages into the page tree; all idempotent).
2. Ensure the main site exposes the internal API and set `MAIN_SITE_INTERNAL_URL` / `INTERNAL_SERVICE_TOKEN` on both sides.
3. Open **System → AI Site Builder** (`/admin/site-builder/prompts`): pick an industry template (capability-gated ones appear only when the corresponding plugin is enabled), type a requirement, click *Preview* to review the plan, then *Execute* to generate into draft.
4. Browse the draft in the preview page (multi-page via the page selector) and fine-tune with the in-place editor or style presets, then *Publish* (a version snapshot is taken automatically).
5. Roll back any time from the version history (restore → draft → edit → publish again).

## Extending the plugin

| To add… | Touch |
|---------|-------|
| New industry template | New `prompts/<id>.yml` following the existing schema (identifier/name/defaults/pages/documents/prompts); auto-seeded on init |
| New style preset | `style_presets.py`: `_preset(...)` entry + keywords in `resolve_style_preset()` |
| New capability / block type | Have the provider plugin declare `site_capabilities` in its `plugin.json`; add the mapping in `capabilities.CAPABILITY_BLOCKS` and a matching partial under `templates/themes/builtin/blocks/` (and any new type into `_SUPPORTED_BLOCKS` if it should be theme-agnostic). For a new *control* block also register it in `block_schemas.py` (`BASE_CONTROL_BLOCK_TYPES`) and add its enum/limit/URL rules to `sanitize_sections()`. |
| Custom theme | New `templates/themes/<id>/` dir with `theme.json` + `layout.html` + `index.html` (+ optional `page-*.html`, `blocks/*`); switch via `tokens.theme.active_theme_id` |
| Block render hook from another plugin | `register_block_partial(type, path)` or `register_block_renderer(type, 'pkg.mod.fn')` (aligns with `add_filter('block/<type>')` semantics) |

## Changelog

See [CHANGELOG.md](./CHANGELOG.md). `plugin.json` is at `2.8.0`; its *Unreleased* sections describe the 2026-09 control-blocks upgrade (Phase 1), the themed-rendering kernel, style presets, capability linking and the page tree (M1) that landed on `master` ahead of release.
