# AI Site Builder (site_builder)

## Overview

AI Site Builder is an LLM-driven website builder: prompt templates, site tasks, and unified design tokens. It is deliberately decoupled from the core system (independent PostgreSQL schema) and communicates with the main site via the internal base URL (`internal_base_url_env`) and internal token (`internal_token_env`).

## Features

- **Prompt templates**: reusable LLM site-generation prompt presets
- **Site tasks**: asynchronous site generation/publish tasks
- **Design tokens**: unified design token system shared with the main site
- **Hooks**: provides `site_builder.build`, `site_builder.publish`, `site_settings.tokens`

## Configuration

| Key | Default | Description |
|-----|---------|-------------|
| `db_url_env` | `SITE_BUILDER_DB_URL` | Env var for the plugin database URL |
| `database_name` | `site_builder` | Database name |
| `schema` | `site_builder` | Dedicated PostgreSQL schema |
| `internal_base_url_env` | `MAIN_SITE_INTERNAL_URL` | Internal main-site base URL |
| `internal_token_env` | `INTERNAL_SERVICE_TOKEN` | Internal service auth token env var |

## Permissions

`admin:access`, `database:write`, `network:request`, `file:write`
