# Search Hub / 检索中枢

> VeroRun 通用检索接入层：将多类外部搜索、网页抓取与证据归一化能力统一为 `EvidenceBundle`。
>
> The unified search gateway for VeroRun: it compiles structured search intents, calls external providers, and returns normalized evidence bundles.

---

## 中文

### 1. 插件定位

Search Hub 是 VeroRun 的 L1/L2 检索基础设施，位于 `plugins/search_hub/`。它不直接实现搜索引擎，而是负责：

1. 接收 Agent、插件或 HTTP 客户端的结构化检索意图 `SearchIntent`；
2. 按 Provider 能力编译为原生查询、过滤参数和结果数量；
3. 处理限流、缓存、月度预算、内容安全检查与错误翻译；
4. 将不同供应商的响应归一化为 `EvidenceBundle`；
5. 通过事件、适配器和 MCP 工具，把证据提供给其他 Agent 或推理引擎。

### 2. 核心能力

- 多 Provider 统一接入：DeepSeek、百度、Brave、Tavily、Exa、Serper、SerpAPI、博查、Yandex、Firecrawl；
- 可配置端点：内置源的 `base_url` 可指向镜像/代理/自托管网关（`GET/PUT /plugin/search-hub/providers[/<name>]`，落 `sh_provider_configs`；仅替换 scheme+host，保留原 path）；
- 自定义源：以 `is_builtin=false` 新增任意兼容 HTTP 搜索源，由 `GenericHttpProvider` 解释，必填 `name`（不得与内置源重名）与 `base_url`，可选 `search_path`（默认 `/search`）、`fetch_path`、`auth_scheme`（`bearer|header|query|api-key|none`）、`auth_header_name`、`auth_query_param`、`key_ref`（统一 Key 池）/`key_value`、`response_mapping`（极简 JSONPath，如 `data.items[0].url`）、`capabilities`、`enabled`；
- 跨源去重：`POST /plugin/search-hub/search/multi`（`providers` ≥ 1，`dedup` 默认 `true`）按归一化 `source_id` 合并，保留 `authority` 最高（同权威分取 `rerank_score` 高者），`meta` 回传 `dedup`/`raw_count`/`merged_count`，被剔除数量记入 `degraded_filters`；插件级默认值 `dedup_across_providers`、`selected_providers`、`default_count` 由 `GET/PUT /plugin/search-hub/config` 维护；
- 统一证据模型：`SearchIntent`、`QueryPlan`、`EvidenceItem`、`SourceEntry`、`EvidenceBundle`；
- HTTP API：`POST /plugin/search-hub/search`、`POST /plugin/search-hub/search/multi`、`POST /plugin/search-hub/fetch`、`GET /plugin/search-hub/providers`、`POST /plugin/search-hub/providers`、`PUT|DELETE /plugin/search-hub/providers/<name>`、`GET|PUT /plugin/search-hub/config`、`GET /plugin/search-hub/health`；
- TTL 缓存：按 Provider 与规范化意图生成稳定缓存键；
- 月度预算：按调用次数与估算费用进行硬闸控制；
- 事件集成：成功后发射 `search.completed`，并监听 `agent.task.completed`；
- MCP Server：通过 stdio 暴露 `search_hub_search` 与 `search_hub_fetch`；
- 证据适配器：面向 Bayes、Butterfly、Chaos 引擎输出不同调用形态；
- 安全控制：Fetch URL 校验、SSRF 防护、PII 预清洗。

### 3. Provider 矩阵

| Provider | 搜索 | 抓取 | 正文/内容 | 时间过滤 | 域名过滤 | 语言过滤 | 鉴权方式 | 代码内单次成本估算 |
|---|---|---|---|---|---|---|---|---:|
| DeepSeek | 是 | 是 | 是 | 是 | 是 | 是 | `Authorization: Bearer` | `$0.00075` |
| 百度 AI 搜索 | 是 | 否 | 原生片段 | 是 | 是 | 否，降级记录 | Bearer 与 `X-Appbuilder-Authorization` | `$0.005`（¥0.036） |
| Brave | 是 | 否 | 否 | 是 | 是 | 是 | `X-Subscription-Token` | `$0.005` |
| Tavily | 是 | 否 | 是 | 仅 news/finance 场景 | 是 | 否，降级记录 | `Authorization: Bearer` | `$0.008` |
| Exa | 是 | 否 | 是 | 是 | 是 | 否，降级记录 | `x-api-key` | `$0.005` |
| Serper | 是 | 否 | 否 | 是 | 是 | 是 | `X-API-KEY` | `$0.0003` |
| SerpAPI | 是 | 否 | 否 | 是 | 是 | 是 | 查询参数 `api_key` | `$0.015` |
| 博查 | 是 | 否 | 是 | 是 | 公开字段未确认，降级记录 | 否，降级记录 | `Authorization: Bearer` | `$0.004` |
| Yandex v2 | 是 | 否 | 否 | 相对时间档 | 否，降级记录 | 是 | `Authorization: Api-Key` | 未声明，回退默认值 |
| Firecrawl | 是 | 是 | 是 | 是 | 是 | 否，降级记录 | `Authorization: Bearer` | `$0.002` |

> 成本仅为代码中的预算估算口径，实际账单以各供应商官方价格、账户套餐和汇率为准。

### 4. 快速开始

#### 配置 API Key

推荐在管理端的插件设置中填写对应 `{provider}_api_key`。解析顺序为：

1. Search Hub 插件配置；
2. VeroRun 统一 `provider_api_keys` 密钥池；
3. 环境变量 `{PROVIDER}_API_KEY`。

本地开发示例：

```powershell
$env:DEEPSEEK_API_KEY = "sk-your-key"
$env:FIRECRAWL_API_KEY = "fc-your-key"
```

#### 执行搜索

```bash
curl -X POST "http://localhost:5000/plugin/search-hub/search" \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: $INTERNAL_SERVICE_TOKEN" \
  -d '{
    "raw_query": "NVIDIA 2025 annual report",
    "entities": ["NVIDIA"],
    "time_window": ["2025-01-01", "2025-12-31"],
    "domains_include": ["nvidia.com"],
    "languages": ["en"],
    "max_results": 10,
    "provider": "deepseek"
  }'
```

#### 抓取网页

```bash
curl -X POST "http://localhost:5000/plugin/search-hub/fetch" \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: $INTERNAL_SERVICE_TOKEN" \
  -d '{"url": "https://example.com/article", "provider": "firecrawl"}'
```

### 5. 文档目录

- [中文白皮书](docs/whitepaper_CN.md)
- [中文用户手册](docs/user-guide_CN.md)
- [中文架构说明](docs/architecture_CN.md)
- [中文 API 参考](docs/api-reference_CN.md)
- [English Whitepaper](docs/whitepaper.md)
- [English User Guide](docs/user-guide.md)
- [English Architecture Guide](docs/architecture.md)
- [English API Reference](docs/api-reference.md)

### 6. 本地测试

在 VeroRun 项目根目录执行：

```powershell
.\.venv\Scripts\python.exe -m unittest discover -s plugins/search_hub/tests
```

最近一次验证结果：170 个测试全部通过，4 个依赖真实 API 或 PostgreSQL 的集成测试按预期跳过。

---

## English

### 1. What Is Search Hub?

Search Hub is VeroRun's unified search and evidence gateway. Rather than exposing each external search API separately, it provides a stable pipeline for agents and plugins:

1. Accept a structured `SearchIntent`;
2. Compile it into the selected provider's native request;
3. Apply rate limiting, caching, budgets, content checks, and error translation;
4. Normalize provider responses into an `EvidenceBundle`;
5. Expose the result through HTTP, MCP, events, and engine-specific adapters.

### 2. Key Features

- Provider abstraction for DeepSeek, Baidu, Brave, Tavily, Exa, Serper, SerpAPI, Bocha, Yandex, and Firecrawl;
- Configurable endpoints: a built-in source's `base_url` can point at a mirror, proxy, or self-hosted gateway (`GET/PUT /plugin/search-hub/providers[/<name>]`, stored in `sh_provider_configs`; only scheme + host are replaced, the original path is kept);
- Custom sources: add any compatible HTTP search source with `is_builtin=false`; it is served by `GenericHttpProvider`. Required: `name` (must not collide with a built-in) and `base_url`. Optional: `search_path` (default `/search`), `fetch_path`, `auth_scheme` (`bearer|header|query|api-key|none`), `auth_header_name`, `auth_query_param`, `key_ref` (central key pool) or `key_value`, `response_mapping` (minimal JSONPath such as `data.items[0].url`), `capabilities`, `enabled`;
- Cross-provider dedup: `POST /plugin/search-hub/search/multi` (`providers` ≥ 1, `dedup` defaults to `true`) merges by normalized `source_id`, keeping the highest `authority` (ties broken by `rerank_score`); `meta` returns `dedup`/`raw_count`/`merged_count` and removed duplicates are recorded in `degraded_filters`. Plugin-level defaults `dedup_across_providers`, `selected_providers`, and `default_count` are managed through `GET/PUT /plugin/search-hub/config`;
- A shared evidence contract based on `EvidenceBundle`;
- HTTP endpoints for search, multi-search, fetch, provider management, plugin config, and health checks;
- Deterministic intent hashing and TTL result caching;
- Monthly call and spend budgets;
- A `search.completed` event and an `agent.task.completed` listener;
- A stdio MCP server with search and fetch tools;
- Adapters for Bayes, Butterfly, and Chaos engines;
- Fetch URL validation, SSRF protection, and PII scrubbing.

### 3. Provider Matrix

| Provider | Search | Fetch | Content | Time Filter | Domain Filter | Language Filter | Authentication | Estimated Cost per Call |
|---|---|---|---|---|---|---|---|---:|
| DeepSeek | Yes | Yes | Yes | Yes | Yes | Yes | `Authorization: Bearer` | `$0.00075` |
| Baidu AI Search | Yes | No | Native snippets | Yes | Yes | No; recorded as degradation | Bearer and `X-Appbuilder-Authorization` | `$0.005` (¥0.036) |
| Brave | Yes | No | No | Yes | Yes | Yes | `X-Subscription-Token` | `$0.005` |
| Tavily | Yes | No | Yes | news/finance only | Yes | No; recorded as degradation | `Authorization: Bearer` | `$0.008` |
| Exa | Yes | No | Yes | Yes | Yes | No; recorded as degradation | `x-api-key` | `$0.005` |
| Serper | Yes | No | No | Yes | Yes | Yes | `X-API-KEY` | `$0.0003` |
| SerpAPI | Yes | No | No | Yes | Yes | Yes | `api_key` query parameter | `$0.015` |
| Bocha | Yes | No | Yes | Yes | Not confirmed in public docs; degraded | No; recorded as degradation | `Authorization: Bearer` | `$0.004` |
| Yandex v2 | Yes | No | No | Relative periods | No; recorded as degradation | Yes | `Authorization: Api-Key` | Not declared; default fallback |
| Firecrawl | Yes | Yes | Yes | Yes | Yes | No; recorded as degradation | `Authorization: Bearer` | `$0.002` |

> These costs are budget estimates embedded in the code. Actual invoices depend on provider pricing, account plans, and exchange rates.

### 4. Quick Start

#### Configure an API Key

Configure `{provider}_api_key` on the plugin settings page. Keys are resolved in this order:

1. Search Hub plugin configuration;
2. VeroRun's central `provider_api_keys` pool;
3. The `{PROVIDER}_API_KEY` environment variable.

For local development:

```powershell
$env:DEEPSEEK_API_KEY = "sk-your-key"
$env:FIRECRAWL_API_KEY = "fc-your-key"
```

#### Search

```bash
curl -X POST "http://localhost:5000/plugin/search-hub/search" \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: $INTERNAL_SERVICE_TOKEN" \
  -d '{
    "raw_query": "NVIDIA 2025 annual report",
    "entities": ["NVIDIA"],
    "time_window": ["2025-01-01", "2025-12-31"],
    "domains_include": ["nvidia.com"],
    "languages": ["en"],
    "max_results": 10,
    "provider": "deepseek"
  }'
```

#### Fetch

```bash
curl -X POST "http://localhost:5000/plugin/search-hub/fetch" \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: $INTERNAL_SERVICE_TOKEN" \
  -d '{"url": "https://example.com/article", "provider": "firecrawl"}'
```

### 5. Documentation

- [English Whitepaper](docs/whitepaper.md)
- [English User Guide](docs/user-guide.md)
- [English Architecture Guide](docs/architecture.md)
- [English API Reference](docs/api-reference.md)
- [中文白皮书](docs/whitepaper_CN.md)
- [中文用户手册](docs/user-guide_CN.md)
- [中文架构说明](docs/architecture_CN.md)
- [中文 API 参考](docs/api-reference_CN.md)

### 6. Testing

Run from the VeroRun repository root:

```powershell
.\.venv\Scripts\python.exe -m unittest discover -s plugins/search_hub/tests
```

Latest verification: 170 tests passed, while 4 integration tests requiring a live provider API or PostgreSQL were skipped as expected.
