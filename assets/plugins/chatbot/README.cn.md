# AI Advisor (chatbot)

> 中文文档。英文事实源见同目录 `README.md`。两份内容同源，最后对齐版本：**v1.7.0**。

## 概述

AI Advisor 是 VeroRun 平台的全站 AI 顾问 / 智能客服插件。它自带**可嵌入的前端组件**与
**匿名可用的公开对话端点**，因此可以被注入到任意页面（一行 `<script>` 即可），
而不依赖任何硬编码在主题模板里的实现。

数据层使用同一 PostgreSQL 实例下的独立 schema `chatbot`，自包含配置、会话与 Agent 注册；
需要工单与知识库时跨读主库。对话能力统一收敛在 `service.answer()` 一个引擎里，
Web 组件、Telegram、LINE、Site Builder 店面、管理端五条链路共用同一份实现。

## 功能特性

- **多形态前端组件**：`bubble`（FAB + 弹层）/ `drawer`（右侧抽屉）/ `inline`（嵌入区块）/
  `fullpage`（整页）。支持拖拽 + 位置记忆、折叠最小化、未读角标、SSE 流式打字机、
  快捷提问、引用来源、CSAT 星级、中英双语、深浅色、Shadow DOM 样式隔离、键盘可达性。
- **知识库接地（RAG）**：复用平台 `agent_matrix.rag_retriever`（向量 + 关键词 + RRF 融合），
  回答附引用来源，并对模型施加"资料未覆盖必须明说、不得编造"的约束；检索失败自动降级。
- **服务端多轮会话**：`chatbot_threads` / `chatbot_turns` 两级持久化，`max_history`
  在服务端真正生效，刷新不丢上下文。
- **确定性转人工**：关键词命中（ASCII 走词边界、CJK 走子串、最长优先）或连续失败达阈值
  即转人工，命中关键词时不调用 LLM；自动建单并把 `ticket_id` 回写会话。
- **匿名可用**：访客令牌（HMAC 签名）+ 滑动窗口限流，无需登录即可对话、转人工、评分。
- **多渠道**：Web 组件、Telegram Bot Webhook、LINE Messaging Webhook，统一走同一引擎，
  会话按 `channel` 归因。
- **运营面板**：今日概览、意图 / 情绪分布、热门问题、座席绩效、对话质检、Agent Copilot。
- **配置单一事实源**：所有读写经 `config_store`，类型转换 + 范围校验 + 白名单 +
  半写保护；管理台改完最迟 30s 全站生效（同进程立即生效）。

## 架构

```
前端（任意页面）                      Telegram / LINE
  一行 <script> 嵌入                        webhook
        │                                      │
        ▼                                      ▼
  widget.js ──SSE──► /plugins/chatbot/*   /api/v1/channels/*
  （四形态组件）        （公开蓝图 public_api）  （channels/router）
                              │                  │
                              └───────┬──────────┘
                                      ▼
                          service.answer()  ← 唯一对话引擎
                    ┌──────────┬──────┴──────┬────────────┐
                    ▼          ▼             ▼            ▼
              config_store  retrieval     threads     escalation
              （配置事实源）（RAG 接地）（会话态/转人工）（建单）
                    │          │             │            │
                    ▼          ▼             ▼            ▼
             chatbot schema  平台 RAG   chatbot schema   主库 user_tickets
```

**设计原则**

- **最大化复用平台缝，内核零改动**：路由经 `register_routes()` → `mount_all_routes()`
  自动挂载（禁用时 `_plugin_gatekeeper` 自动 404）；跨模块调用经
  `shared/plugin_access.PLUGIN_TOUCHPOINTS`；RAG 经 `agent_matrix.rag_retriever`；
  建表经插件独立 `migrations/`。
- **独立 schema + 主库只读**：插件不污染主库，只在建工单时写 `user_tickets`。
- **一份引擎，多个入口**：任何渠道都不得自行拼 prompt / 自行判定转人工。

## 目录结构

```
chatbot/
├── README.md                  # 英文文档（平台默认抓取名）
├── README.cn.md               # 中文文档（平台优先抓取名）
├── CHANGELOG.md
├── plugin.json                # 元数据 / 权限 / settings_schema / metadata.agents
├── __init__.py                # 插件入口：setup/activate/register_routes/钩子注册
├── config_store.py            # 运行时配置单一事实源（WP-A）
├── threads.py                 # 服务端会话态 + 转人工判定（WP-E）
├── retrieval.py               # RAG 接地 + JSON 健壮解析（WP-D）
├── guard.py                   # 访客令牌 / 限流 / 验签 / PII 脱敏（WP-F）
├── service.py                 # 统一对话引擎（WP-B）
├── escalation.py              # [TICKET_CREATE] 解析与建单
├── public_api.py              # 公开蓝图 /plugins/chatbot/*
├── routes.py                  # 管理端 API + 渠道 webhook
├── models.py                  # 独立库连接、建表、Agent 注册、主库迁移
├── stats.py                   # 统计、质检、Copilot
├── channels/router.py         # Telegram / LINE 收发
├── prompts/sub_chatbot_prompt.md
├── migrations/                # 版本化 SQL 文档
├── templates/
│   ├── admin_chatbot.html     # 管理后台页面
│   └── widget/                # 前端组件（chatbot-widget.js / .css）
├── i18n/                      # zh-CN.yml / en.yml
└── tests/                     # 离线单测（193 Python + 26 node）
```

## 安装与启用

前提：VeroRun ≥ 0.10.0；PostgreSQL；可用的 LLM Provider；
（启用 RAG 时）`plugins/_base/embeddings` 与 pgvector 可用，缺失会自动降级为纯关键词检索。

1. 将 `chatbot` 目录置于 `plugins/` 下，确认 `plugin.json` 的 `enabled` 为 `true`。
2. **设置环境变量 `CHATBOT_VISITOR_SECRET`**（多进程部署必须；admin / main_site /
   site_builder 各为独立进程，未统一密钥时 A 进程签发的访客令牌在 B 进程验不过）。
3. 重启应用。插件自动：建 schema 与全部表（含 `chatbot_threads` / `chatbot_turns`）、
   写默认配置、从主库幂等迁移历史数据、把历史遗留在主库的配置吸收进事实源、
   注册 Agent 能力与两个 filter 钩子、挂载三个蓝图。
4. 管理后台 → “AI & Content” → “AI Advisor” 配置文案、形态、RAG、限流与转人工规则。

启用渠道（可选）：在 IM Gateway 插件配置 Telegram / LINE 凭证，并**务必**设置
`TELEGRAM_SECRET_TOKEN` / `LINE_CHANNEL_SECRET` —— 未设置时 webhook 一律返回 403
（fail-closed），不会静默放行。

## 前端接入

在任意页面加一行（`GET /plugins/chatbot/embed` 可直接取到当前配置对应的代码）：

```html
<script src="/plugins/chatbot/widget.js" defer
        data-mode="bubble" data-position="bottom-right" data-theme="auto"></script>
```

- 需要手动控制时：`data-autoboot="false"`，然后
  `VeroRunAdvisor.init({ mode: 'drawer', container: '#help', lang: 'en' })`。
- 运行时 API：`init / boot / setMode / toggle / clear / destroy`，实例挂在
  `VeroRunAdvisor.instances`。
- `inline` 形态用 `data-container="#selector"` 指定宿主容器；`fullpage` 建议单独一个路由页。
- 组件不依赖宿主页面的任何全局变量与样式，Shadow DOM 装载时样式天然隔离。

## 配置说明

全部配置项的**唯一事实源**是 `chatbot.plugin_configs`；管理台写入后会镜像一份到
`plugin_registry.config` 供商店 / 元数据使用，但运行时永不从镜像读取。

| 配置项 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `enabled` | bool | true | 总开关。关闭后公开端点返回 503，组件不渲染，插件路由被门卫 404 |
| `auto_escalate` | bool | true | 转人工时是否自动建工单 |
| `title` / `subtitle` | string | AI Advisor | 组件标题 / 副标题 |
| `welcome_message` | string | — | 欢迎语 |
| `help_hint` | string | — | 帮助提示 |
| `avatar_url` | string | "" | 头像（支持媒体库选择与上传） |
| `agent_id` | string | chat_assistant | 绑定 `agent_registry` 中的 Agent（**v1.7.0 起真正生效**） |
| `max_history` | int 1–50 | 20 | 服务端保留的对话轮数 |
| `float_button_text` | string | AI Advisor | FAB 文案 |
| `handoff_keywords` | JSON array | 中英各 9–10 个 | 转人工关键词（ASCII 词边界 / CJK 子串 / 最长优先） |
| `handoff_max_fails` | int 1–10 | 3 | 连续失败阈值 |
| `widget_mode` | enum | bubble | bubble / drawer / inline / fullpage |
| `widget_position` | enum | bottom-right | bottom-right / bottom-left |
| `widget_theme` | enum | auto | auto / light / dark |
| `quick_replies` | JSON array | [] | 快捷提问（最多展示 4 条） |
| `embed_key` | string | "" | 公开端点的可选共享密钥；留空则不校验 |
| `rag_enabled` | bool | true | 是否启用知识库接地 |
| `rag_top_k` | int 1–20 | 5 | 检索条数（上限对齐平台检索器） |
| `rag_min_score` | float 0–1 | 0.0 | 融合分下限 |
| `rate_limit_per_min` | int 1–600 | 10 | 每访客每分钟对话次数上限 |
| `visitor_token_secret` | string | "" | 留空则用环境变量 `CHATBOT_VISITOR_SECRET` |

写入行为：任一键非法（越界 / 类型错 / 不在白名单）→ **整批拒绝**，不留半更新状态。

## API 端点

### 公开端点（匿名可用，前缀 `/plugins/chatbot`）

| 方法 | 路径 | 鉴权 | 说明 |
|---|---|---|---|
| GET | `/config` | 无（可选 `embed_key`） | 组件启动配置，**字段白名单**，不含运营规则与密钥 |
| POST | `/session` | 无 | 签发访客令牌 + 服务端 `thread_id`（6 次/分/IP） |
| POST | `/messages` | 访客令牌或 JWT | 对话；`stream` 默认 true 走 SSE，false 返回 JSON |
| POST | `/actions/escalate` | 访客令牌或 JWT | 转人工建单 |
| POST | `/actions/csat` | 访客令牌或 JWT | 满意度评分 1–5 |
| GET | `/widget.js` `/widget.css` | 无 | 组件资产（ETag / 304 / nosniff） |
| GET | `/embed` | 无 | 生成一行嵌入代码（`?format=text` 返回纯文本） |

SSE 事件：`start` → `token`×N → `done`（含 `reply` / `thread_id` / `handoff` /
`ticket_id` / `references`），异常时 `error`（含 `code`）。
`[TICKET_CREATE]` 内部标记与工单 JSON 在流式下发时被抑制，不会泄漏给访客。

### 管理端（前缀 `/admin/chatbot`）

| 方法 | 路径 | 鉴权 | 说明 |
|---|---|---|---|
| POST | `/chat` | admin | 对话（走统一引擎，保持 `data.reply` 契约） |
| GET / POST | `/settings` | admin | 读写全量运行时配置 |
| GET / POST | `/handoff_rules` | admin | 读写转人工规则（与会话配置同源） |
| GET | `/stats` `/hot_topics` `/agent_performance` | admin | 运营面板数据 |
| POST | `/log_session` `/qa_check` `/copilot_suggest` `/csat` `/escalate` | login | 兼容既有前端调用 |

### 渠道 Webhook（前缀 `/api/v1/channels`）

| 方法 | 路径 | 鉴权 | 说明 |
|---|---|---|---|
| POST | `/telegram/webhook` | `X-Telegram-Bot-Api-Secret-Token`，**fail-closed** | 未配置 secret 一律 403 |
| POST | `/line/webhook` | `x-line-signature`（HMAC-SHA256），**fail-closed** | 同上 |

## 依赖与平台缝

| 依赖 | 用途 |
|---|---|
| `plugins._base.db` | 独立 schema 连接池 |
| `agent_matrix.engine` / `intent` / `rag_retriever` / `models` | LLM、意图情绪、RAG、能力注册 |
| `plugin_manager.hooks` / `base` / `logger` | 钩子、生命周期、日志 |
| `shared.plugin_access` | 核心侧调用插件的唯一受控入口（已登记 6 个触点） |
| `auth-center.models` / `services.jwt_service` | 主库工单、登录态解析（**调用期**导入） |
| `plugins.im_gateway` | Telegram / LINE 凭证 |

提供的 Hook（filter，回调首参为 `value`）：

| Hook | 语义 |
|---|---|
| `chatbot/config` | 传入 dict 覆盖项 → 返回合并后的运行时配置 |
| `chatbot/chat` | 传入消息文本或 `{'message': ...}` → 返回 `service.answer()` 结果 dict |

Agent 注册：名称 `Advisor Agent`、标识 `chat_assistant`、角色 `sub`、领域 `chatbot`、
能力 `chatbot.faq` / `chatbot.ticket` / `chatbot.human_handoff`，
提示词取自 `prompts/sub_chatbot_prompt.md`（由 `plugin.json` 的 `metadata.agents` 声明驱动）。

## 已知限制

- 统计面板口径仍为“今日”，未提供时间范围筛选与导出；`trend` 数据已计算但后台未绘制。
- 无座席实时接管工作台：`copilot_suggest` 只给建议，工单 `assigned_to` 建单时为 NULL，
  无技能组路由 / 排队 / SLA。
- 渠道仅 Telegram 与 LINE，无 WhatsApp / 微信 / 企微；跨渠道身份未归一。
- 无多模态（图片 / 语音 / 文件）、无主动触达、无 A/B 与离线评测集。
- 限流为**进程内**滑动窗口：多进程部署时各进程独立计数，等效阈值约为
  `rate_limit_per_min × 进程数`；需要全局精确限流时应改接网关或 Redis。
- `POST /api/v1/chat`（main_site 自带、免登录）仍是独立实现，本次只修正了它的
  转人工规则读取源；建议后续切到 `chatbot/chat` 钩子或 `service.answer()` 触点。

## 开发与测试

```bash
# Python 离线单测（193 条；全部以桩替换 DB / LLM / 主库 / 出站 HTTP，不连任何真实服务）
python -m unittest discover -s plugins/chatbot/tests -t .

# 前端组件纯逻辑单测（26 条，node ≥ 18，无 jsdom 依赖）
node --check plugins/chatbot/templates/widget/chatbot-widget.js
node --test plugins/chatbot/tests/widget/core.test.js
```

约定：新增配置项必须同时改 `config_store.RUNTIME_KEYS` / `DEFAULTS` 与
`plugin.json` 的 `config` / `settings_schema`；新增表必须同时改
`threads.py`/`models.py` 的幂等建表与 `migrations/` 的版本化 SQL。

## 许可证

本插件为 VeroRun 平台的一部分，遵循平台统一的许可证协议。
