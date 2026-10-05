# Social Push (social_push)

## Overview

Social Push is VeroRun's multi-platform social media content publishing plugin. It supports seven platforms: WeChat Official Account, Weibo, Toutiao, Twitter/X, LinkedIn, Reddit, and Telegram Channel.

**As of v2.0.0**, all social-media OAuth, account management, publishing, and token refresh have been migrated back from `im_gateway` — social_push is now the **single owner** of social media business, and `im_gateway` has returned to pure IM. The plugin stores publish history and channel accounts in a dedicated PostgreSQL schema `social_push`.

The plugin uses a Provider pattern for international platforms (Twitter, LinkedIn, Reddit, Telegram); domestic platforms (WeChat, Weibo, Toutiao) reuse their respective push services. AI copywriting and AI image generation go through the platform-wide shared LLM service (`ai_content_generator`) and are decoupled from publishing channels.

## Features

- **Seven platforms**: WeChat Official Account, Weibo, Toutiao, Twitter/X, LinkedIn, Reddit, Telegram Channel
- **OAuth & account management** (v2.0.0): channel accounts stored in the plugin's own `channel_accounts` table (JSON format, including `__app__` row for per-platform app credentials); OAuth blueprint mounted under `/admin/channels/oauth`; admin UI adds **Accounts** and **OAuth Connections** tabs
- **Token auto-refresh** (v2.0.0): APScheduler job refreshes expired access tokens per platform
- **Idempotent data migration** (v2.0.0): historical channel accounts from `im_gateway` migrated on first enable (batch B1)
- **Provider pattern**: international platforms extend via adapters under `providers/` with a unified interface
- **AI copywriting**: Tongyi Qianwen based content generation (shared LLM service)
- **AI cover image**: Tongyi Wanxiang based cover image generation (shared LLM service)
- **Publish history**: complete publish log with per-platform filtering and pagination
- **CMS article import**: import already-published CMS articles into the social editor
- **Publish status query**: WeChat publish status callback support
- **Config detection**: automatically detects each platform's config status
- **Market awareness**: distinguishes domestic/international markets via `DEPLOY_MARKET` env var
- **Dedicated database**: PostgreSQL schema `social_push` with publish logs + channel accounts

## Architecture

```
+--------------------------------------------------------------+
|                    Admin Dashboard UI                         |
+--------------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------------+
|                     Route layer (routes.py)                   |
|  /admin/social/*                                             |
|  +-- /check-config     Platform config detection             |
|  +-- /content-types    Content type query                    |
|  +-- /generate         AI copywriting generation             |
|  +-- /generate-image   AI image generation                   |
|  +-- /publish          Multi-platform publishing             |
|  +-- /publish-status   WeChat publish status query           |
|  +-- /history          Publish history query                 |
|  +-- /import-from-cms  CMS article import                    |
|  +-- /history/<id>     Delete publish record                 |
+--------------------------------------------------------------+
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
+------------------+ +------------------+ +------------------+
| _publish_wechat  | | _publish_weibo   | | _publish_toutiao |
| (WeChat draft +  | | (Weibo publish)  | | (Toutiao publish)|
|  publish)        | |                  | |                  |
+------------------+ +------------------+ +------------------+
                              |
                              v
              +-------------------------------+
              | _publish_via_provider()       |
              | (Twitter/LinkedIn/Reddit/     |
              |  Telegram unified entry)      |
              +-------------------------------+
                              |
                              v
+--------------------------------------------------------------+
|                   Provider layer (providers/)                 |
|  +-- base.py               Provider abstract base            |
|  +-- twitter.py            Twitter/X Provider                |
|  +-- linkedin.py           LinkedIn Provider                 |
|  +-- reddit.py             Reddit Provider                   |
|  +-- telegram_channel.py   Telegram Channel Provider         |
+--------------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------------+
|                       Data layer (models.py)                  |
|  PG Schema: social_push                                       |
|  +-- social_push_logs    Publish history table               |
+--------------------------------------------------------------+
                              |
                              v
+--------------------------------------------------------------+
|                   auth-center service layer                   |
|  +-- services/wechat_push_service.py   WeChat push service    |
|  +-- services/weibo_service.py         Weibo push service     |
|  +-- services/toutiao_service.py       Toutiao push service   |
|  +-- services/ai_content_generator.py  Shared AI service      |
+--------------------------------------------------------------+
```

**AI capabilities are decoupled from publishing channels**:

```
Creation tools (ai_capabilities)          Channels (platforms)
+---------------------------+      +---------------------------+
| AI copywriting (Qianwen)  |      | WeChat OA (wechat)        |
| AI cover image (Wanxiang) |      | Weibo (weibo)             |
| Via shared LLM service    |      | Toutiao (toutiao)         |
| (ai_content_generator)    |      | Twitter/X (twitter)       |
+---------------------------+      | LinkedIn (linkedin)       |
                                   | Reddit (reddit)           |
                                   | Telegram (telegram)       |
                                   +---------------------------+
```

## Directory Structure

```
social_push/
+-- README.md                    # Plugin documentation (English)
+-- README_CN.md                 # Plugin documentation (Chinese)
+-- plugin.json                  # Plugin metadata configuration
+-- __init__.py                  # Plugin entry, blueprint & Hook registration
+-- models.py                    # Publish logs data model
+-- models_accounts.py           # Channel accounts data model (v2.0.0)
+-- routes.py                    # Admin API routes (publish, history, AI generation)
+-- routes_oauth.py              # OAuth callback routes (v2.0.0, /admin/channels/oauth)
+-- scheduler.py                 # Token auto-refresh scheduler (v2.0.0)
+-- crypto.py                    # Credential encryption helpers
+-- providers/
|   +-- __init__.py              # Provider registration & factory functions
|   +-- base.py                  # Provider abstract base class
|   +-- twitter.py               # Twitter/X Provider
|   +-- linkedin.py               # LinkedIn Provider
|   +-- reddit.py                 # Reddit Provider
|   +-- telegram_channel.py       # Telegram Channel Provider
+-- i18n/
|   +-- en.yml                   # English internationalization (242 keys)
|   +-- zh-CN.yml                # Chinese internationalization
+-- templates/
    +-- admin_socialpush.html    # Admin dashboard page template
```

## Install & Enable

### Prerequisites

- VeroRun platform version >= 0.10.0
- API credentials for the target platforms (e.g. WeChat Official Account AppID/AppSecret, Weibo App Key, etc.)
- DashScope API Key (for AI copywriting and cover image generation)
- PostgreSQL database

### Install Steps

1. Place the `social_push` directory under `plugins/`
2. Restart the application; the plugin auto-registers on startup and:
   - Creates the PostgreSQL schema `social_push`
   - Initializes the `social_push_logs` table
   - Idempotently migrates historical publish records from the main database
3. Configure each platform in the admin dashboard under "Publishing" > "Social Push"
4. Use the `check-config` endpoint to verify the config status of each platform

### Environment Variables

| Variable | Description |
|----------|-------------|
| `DEPLOY_MARKET` | Deployment market: `cn` (domestic, only shows domestic platforms) / `intl` (international, shows all platforms) |

## Configuration

As of v2.0.0, channel accounts and OAuth credentials are managed in the plugin's own `channel_accounts` table (JSON format), not in auth-center's `system_config`. The admin UI provides two dedicated tabs: **Accounts** and **OAuth Connections** for connect / credential management / token refresh / revoke / test publish.

App-level credentials (per-platform AppID/AppSecret) are stored in a special row with `account_key='__app__'`.

### Environment Variables

| Variable | Description |
|----------|-------------|
| `DEPLOY_MARKET` | Deployment market: `cn` (domestic only) / `intl` (all platforms) |

### AI Capabilities

| Capability | Config |
|------------|--------|
| AI copywriting (Tongyi Qianwen) | via shared `ai_content_generator` service |
| AI cover image (Tongyi Wanxiang) | via shared `ai_content_generator` service |

## API Endpoints

### Config Detection

| Method | Path | Description |
|--------|------|-------------|
| GET | `/admin/social/check-config` | Detects config status of each platform and AI capability |

### AI Content Generation

| Method | Path | Description |
|--------|------|-------------|
| POST | `/admin/social/generate` | AI copywriting generation (supports topic, content_type, temperature) |
| POST | `/admin/social/generate-image` | AI image generation (supports prompt, title, cover mode) |

### Content Types

| Method | Path | Description |
|--------|------|-------------|
| GET | `/admin/social/content-types` | Queries the content types supported by each platform |

### Publishing

| Method | Path | Description |
|--------|------|-------------|
| POST | `/admin/social/publish` | Publishes content to the given platforms (multi-platform in one request) |
| GET | `/admin/social/publish-status/<publish_id>` | Queries WeChat publish status |

### Publish History

| Method | Path | Description |
|--------|------|-------------|
| GET | `/admin/social/history` | Paginated publish history (supports platform filtering) |
| DELETE | `/admin/social/history/<id>` | Deletes a publish history record |

### CMS Import

| Method | Path | Description |
|--------|------|-------------|
| GET | `/admin/social/import-from-cms` | Lists published CMS articles for import |

### Publish Request Body Example

```json
{
  "title": "Article title",
  "body": "Article body",
  "body_html": "<h1>HTML body</h1>",
  "summary": "Summary",
  "author": "admin",
  "cover_image_url": "/static/uploads/cover.jpg",
  "platforms": ["wechat", "weibo", "twitter"],
  "auto_publish": false
}
```

## Platform Content Types

| Platform | Supported content types |
|----------|-------------------------|
| WeChat Official Account | article, announcement, promotion |
| Weibo | weibo |
| Toutiao | article |
| Twitter/X | tweet |
| LinkedIn | article, post |
| Reddit | link, text |
| Telegram | message |

## Dependencies

### Internal Dependencies

| Dependency | Purpose |
|------------|---------|
| `plugins._base.db` | Plugin base database connection module |
| `auth-center.models` | Main-db reads (system_config, cms_posts) |
| `auth-center.services.wechat_push_service` | WeChat push (`create_draft`, `submit_publish`, `upload_article_image`) |
| `auth-center.services.weibo_service` | Weibo publish (`publish_weibo`) |
| `auth-center.services.toutiao_service` | Toutiao publish (`publish_article`) |
| `auth-center.services.ai_content_generator` | Shared AI service (`generate_article`, `generate_image`, `generate_cover_image`) |

### External Dependencies

| Dependency | Purpose |
|------------|---------|
| WeChat Official Account API | Draft creation & publishing |
| Weibo Open Platform API | Weibo publishing |
| Toutiao API | Toutiao article publishing |
| Twitter/X API | Tweet publishing |
| LinkedIn API | Article/update publishing |
| Reddit API | Post publishing |
| Telegram Bot API | Channel message publishing |
| DashScope (Tongyi Qianwen) | AI copywriting generation |
| DashScope (Tongyi Wanxiang) | AI cover image generation |

### Provided Hooks

| Hook identifier | Description |
|-----------------|-------------|
| `social_push/publish` | Publishes content to social media platforms |

## Menu Group

- **Publishing** - Social Push

## Extension Guide

### Adding a New Social Media Platform

1. Create a new Provider file under `providers/` (e.g. `facebook.py`)
2. Inherit `providers.base.BaseSocialProvider` and implement the `publish()` and `get_config_fields()` methods
3. Register the new platform in the `get_provider()` factory in `providers/__init__.py`
4. Add a route branch in `_publish_to_platform()` in `routes.py`

```python
# providers/facebook.py example
from .base import BaseSocialProvider

class FacebookProvider(BaseSocialProvider):
    platform_id = 'facebook'
    platform_name = 'Facebook'

    def get_config_fields(self):
        return [
            {'key': 'facebook_app_id', 'label': 'App ID', 'type': 'text'},
            {'key': 'facebook_app_secret', 'label': 'App Secret', 'type': 'password'},
            {'key': 'facebook_page_token', 'label': 'Page Access Token', 'type': 'password'},
        ]

    def publish(self, title, body, summary, image_url, link_url, config):
        # Implement the Facebook Graph API publishing logic
        ...
```

## License

This plugin is part of the VeroRun platform and follows the platform's unified license agreement.

## Change Log

### v1.2.0 (2026-08-06)

Audit-driven fixes (19 findings fully resolved):

**Blocking**
- Reddit publishing now works end-to-end: the subreddit is passed through from the publish form to the Reddit provider
- `_get_international_providers()` signature aligned with its call site so provider configs are passed, making `is_configured()` accurate
- Version unified to 1.2.0 across `plugin.json` / `__init__.py`

**High**
- LinkedIn `shareMediaCategory` now switches between `ARTICLE` / `NONE` based on whether a link is provided
- tweepy response accessed via `resp.data.id` instead of `resp.data['id']`
- Removed the external `get_market` import — the market is read directly from the `DEPLOY_MARKET` env var
- `category` normalized to the v1.4 standard enumeration (`social`)

**Medium**
- `settings_schema` fully declared with valid JSON Schema (20 config items)
- Thread-safe per-thread DB connections (`threading.local`)
- `created_at` migrated to `TIMESTAMPTZ` with an idempotent ALTER for legacy records
- `content_factory` availability check on the admin page (AI buttons disabled when unavailable)
- Missing "Import from CMS" i18n keys added (en / zh-CN)
- v1.4 store compliance: `dashboard.stats` declared + `get_dashboard_stats()` implemented

**Low**
- Toutiao success message quote typo fixed
- `sys.path` pollution removed from the plugin entry
- Non-standard `enabled` field removed from `plugin.json`
- Telegram sends photos via `sendPhoto` when an image URL is provided
- Minor i18n string spacing fixes
