# Mini App Builder (mini_app_builder)

The Mini App Builder plugin provides **natural-language & template mini-program generation**, **multi-platform code output**, **deployment & preview**, **publishing** (WeChat / Douyin), and **encrypted developer-credential management**. It merges the legacy `dev_accounts` plugin and depends on `oauth_config`.

The plugin writes to a dedicated PostgreSQL schema and keeps its own task queue for generation/deploy.

---

## English

## Features

### 1. Generate from Plain Language

`POST /admin/site-builder/mini-app/generate`

- **AI generation** — multi-round LLM pipeline (Parse → Brand → per-page) that turns a prompt into a project plan and scaffolds real code.
- **Knowledge injection** — local knowledge retrieved via RAG (`scope='user'`) is injected into the generation prompt, so generated pages reflect your real business content.
- **Template generation** — four built-in templates: `shop`, `community`, `brand`, `service` (`prompts/*.yml`).

### 2. Multi-Platform Output

One project produces source for every enabled platform:

| Platform | Runtime | Support |
|----------|---------|---------|
| WeChat | WXML / WXSS / JS | generate + publish |
| Douyin / Toutiao | TTML / TTSS / JS | generate + publish |
| LINE | HTML | generate + deploy/preview |
| Telegram | HTML | generate + deploy/preview |
| WhatsApp | HTML | generate + deploy/preview |

Output is packaged with `MiniAppPackager` and downloaded as a ready-to-import project. Generation runs in a task queue (`max_concurrent_tasks`) with status polling.

### 3. Local Knowledge Base (RAG) at Runtime

- `POST /chat/stream` — SSE streaming chat with RAG context; rate-limited for anonymous callers (sliding window, per IP).
- `GET /knowledge/search?q=&topK=&category=` — standalone retrieval for the `VeroRAG` frontend SDK.
- Retrieval always uses `scope='user'`, isolated from admin `system` knowledge.

### 4. Publishing (v2.4.0+)

- **WeChat** — automatic experience-build upload via `miniprogram-ci` (requires Node.js ≥ 16; degrades to manual-upload guidance when Node is unavailable), audit submit and audit-status polling over HTTP.
- **Douyin / Toutiao** — package upload, audit submit, audit-status polling.
- Every attempt is recorded in `publish_logs` with status `draft → uploading → uploaded → auditing → approved / rejected / failed`.

### 5. Developer Credentials

Encrypted storage (`MINI_APP_ENCRYPTION_KEY`), reused across generation, upload and audit flows. Test connection uses raw credentials to avoid masking issues.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/site-builder/mini-app/projects` | List projects |
| POST | `/admin/site-builder/mini-app/projects` | Create project |
| GET | `/admin/site-builder/mini-app/projects/<id>/versions` | List versions |
| POST | `/admin/site-builder/mini-app/generate` | Generate (AI/template) |
| GET | `/admin/site-builder/mini-app/status/<task_id>` | Task status |
| GET | `/admin/site-builder/mini-app/download/<platform>/<task_id>` | Download package |
| POST | `/admin/site-builder/mini-app/deploy/<platform>` | Deploy |
| GET | `/admin/site-builder/mini-app/preview/<version_id>` | Preview |
| POST | `/admin/site-builder/mini-app/publish/<platform>/upload` | Upload build |
| POST | `/admin/site-builder/mini-app/publish/<platform>/audit` | Submit audit |
| GET | `/admin/site-builder/mini-app/publish/status` | Audit status |
| GET | `/admin/site-builder/mini-app/publish/logs` | Publish logs |

## Configuration

| Key | Default | Description |
|-----|---------|-------------|
| `output_base` | `workspace/mini_apps` | Output directory for generated mini-apps |
| `task_ttl_seconds` | `3600` | Task lifetime before cleanup |
| `max_concurrent_tasks` | `5` | Max concurrent generation tasks |
| `ai_generation_enabled` | `true` | Toggle LLM-assisted generation |
| `encryption_key_env` | `MINI_APP_ENCRYPTION_KEY` | Env var holding the credential encryption key |

## Dependencies

- Plugin: `oauth_config >= 1.0.0` (OAuth login/token management)
- Runtime: Node.js ≥ 16 + `miniprogram-ci` (WeChat auto-upload, optional)

---

## 中文

## 功能特性

### 1. 自然语言生成

接口：`POST /admin/site-builder/mini-app/generate`

- **AI 生成** — 多轮 LLM 流水线（Parse → Brand → 每页），把描述转成项目方案并生成真实代码。
- **知识注入** — 通过 RAG 检索本地知识（`scope='user'`）注入生成提示词，让生成页面贴合真实业务内容。
- **模板生成** — 内置四类模板：`shop`（商城）、`community`（社区）、`brand`（品牌）、`service`（服务）（`prompts/*.yml`）。

### 2. 多平台产出

一个项目为每个启用平台产出代码：

| 平台 | 运行时 | 支持 |
|------|--------|------|
| 微信 | WXML / WXSS / JS | 生成 + 发布 |
| 抖音 / 头条 | TTML / TTSS / JS | 生成 + 发布 |
| LINE | HTML | 生成 + 部署/预览 |
| Telegram | HTML | 生成 + 部署/预览 |
| WhatsApp | HTML | 生成 + 部署/预览 |

产物经 `MiniAppPackager` 打包，下载后即为可直接导入的项目。生成在任务队列中执行（`max_concurrent_tasks`），支持状态轮询。

### 3. 运行时本地知识库（RAG）

- `POST /chat/stream` — SSE 流式对话，携带 RAG 上下文；匿名访客按 IP 滑动窗口限频。
- `GET /knowledge/search?q=&topK=&category=` — 独立检索，供前端 `VeroRAG` SDK 调用。
- 检索固定使用 `scope='user'`，与管理端 `system` 知识隔离。

### 4. 发布链路（v2.4.0+）

- **微信** — 通过 `miniprogram-ci` 自动上传体验版（需 Node.js ≥ 16；Node 不可用时降级为手动上传指引），提审提交与审核状态轮询走 HTTP。
- **抖音 / 头条** — 包上传、提审提交、审核状态轮询。
- 每次发布记录在 `publish_logs` 表，状态流转 `draft → uploading → uploaded → auditing → approved / rejected / failed`。

### 5. 开发者账号

凭证加密存储（`MINI_APP_ENCRYPTION_KEY`），生成、上传、提审全链路复用；连接测试使用原始凭证，避免脱敏导致误判。

## 端点一览

| 方法 | 路径 | 用途 |
|------|------|------|
| GET | `/admin/site-builder/mini-app/projects` | 项目列表 |
| POST | `/admin/site-builder/mini-app/projects` | 创建项目 |
| GET | `/admin/site-builder/mini-app/projects/<id>/versions` | 版本列表 |
| POST | `/admin/site-builder/mini-app/generate` | 生成（AI/模板） |
| GET | `/admin/site-builder/mini-app/status/<task_id>` | 任务状态 |
| GET | `/admin/site-builder/mini-app/download/<platform>/<task_id>` | 下载代码包 |
| POST | `/admin/site-builder/mini-app/deploy/<platform>` | 部署 |
| GET | `/admin/site-builder/mini-app/preview/<version_id>` | 预览 |
| POST | `/admin/site-builder/mini-app/publish/<platform>/upload` | 上传产物 |
| POST | `/admin/site-builder/mini-app/publish/<platform>/audit` | 提交提审 |
| GET | `/admin/site-builder/mini-app/publish/status` | 审核状态 |
| GET | `/admin/site-builder/mini-app/publish/logs` | 发布日志 |

## 配置项

| 键 | 默认值 | 说明 |
|-----|---------|------|
| `output_base` | `workspace/mini_apps` | 生成产物输出目录 |
| `task_ttl_seconds` | `3600` | 任务存活时间（到期清理） |
| `max_concurrent_tasks` | `5` | 最大并发生成任务数 |
| `ai_generation_enabled` | `true` | 是否启用 LLM 辅助生成 |
| `encryption_key_env` | `MINI_APP_ENCRYPTION_KEY` | 凭证加密密钥所在环境变量 |

## 依赖

- 插件：`oauth_config >= 1.0.0`（OAuth 登录 / 令牌管理）
- 运行时：Node.js ≥ 16 + `miniprogram-ci`（微信自动上传，可选）
