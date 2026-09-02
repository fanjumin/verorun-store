# VeroScholar — 引源索骥

VeroScholar 是 VeroRun AI 系统教育版的科研全流程插件，覆盖**选题 · 文献 · 写作 · 审稿**四个核心阶段。

插件严格遵循 [`docs/plugin-standard-v1.7.md`](../../docs/plugin-standard-v1.7.md) 开发：
单库多 Schema（`veroscholar`）、共享连接池、JWT 管理员鉴权、iframe 独立页、
i18n 双语文案、卸载零残留。

## 功能模块

| 模块 | 说明 |
|------|------|
| 多源文献检索 | arXiv / Semantic Scholar / OpenAlex / 中文作品（OpenAlex `language:zh`）多库聚合，DOI 去重（title+year 兜底），`semantic` 与 `semantic_scholar` 别名；支持年份/被引/期刊后置过滤 |
| 论文知识库 | 论文入库、搜索、详情；无 DOI 时按 title+year 兜底去重 |
| 研究项目 | 项目 CRUD、成员管理、论文归类 |
| 阅读笔记 | 论文级笔记标注，可选 embedding 向量（pgvector）供语义检索 |
| 综述生成器 | 触发 DAG 工作流：关键词扩展 → 多源检索 → 确定性去重排序 → 方法分类 → LLM 生成综述 |
| 引用校验 | 综述引用权威注册机构在线验真（Crossref/DataCite）+ 撤稿检测，五态判定（registered/retracted/concern/unregistered/offline），本地库补充标记 |
| 引文导出 | 单篇论文导出 BibTeX / RIS（预印本自动识别为 `@misc`/`TY - RPRT`），供写作引用管理 |
| 论文问答 | 基于论文摘要 + 阅读笔记的 RAG 问答（仅基于给定材料，防幻觉） |
| 摘要翻译 | 论文摘要中英互译（LLM） |
| 相关论文推荐 | 摘要向量余弦相似度推荐相似论文 |
| 标签系统 | 论文级标签（幂等增删）+ 库按标签筛选 |
| 阅读状态 | unread / reading / read 三态，库卡片徽标 + 抽屉切换 |
| 全文检索 | pg_trgm GIN 索引加速 title/abstract 检索（中文友好，无分词器依赖） |
| PDF 全文（B1） | PDF 上传（≤10MB，sha256 幂等）→ pypdf 解析 → 段落分块入库 → 分块浏览；论文问答自动注入 top-3 相关全文摘录（钩子注入，失败静默降级） |
| 3 个子 Agent | Literature Review（high tier）、Experiment Designer（high tier）、Paper Writer（standard tier） |

## 目录结构

```
plugins/veroscholar/
├── __init__.py                  # BasePlugin 生命周期组装
├── models.py                    # 数据层（共享连接池 + search_path 隔离）
├── routes.py                    # Blueprint：3 个页面 + RESTful API + JWT 鉴权
├── workflow.py                  # DAG 节点处理器 + 综述工作流触发
├── services/                    # AI 服务层（问答 / 翻译 / 相关推荐）
├── plugin.json                  # 插件元数据（agents/menu/dashboard/settings）
├── migrations/v1.0.0_init.sql   # schema 建表（全部 IF NOT EXISTS 幂等）
├── migrations/v1.1.1_lib.sql    # pg_trgm 索引 + 标签表 + 阅读状态
├── migrations/v1.2.0_verify.sql # DOI 校验缓存 + 检索策略落库
├── migrations/v1.3.0_fulltext.sql # PDF 全文表（B1，零扩展依赖）
├── fulltext/                    # PDF 全文模块（parser / chunker / qa_ext / routes）
├── adapters/                    # 数据源适配器（base + arxiv + semantic_scholar + openalex + openalex_zh）
├── agents/                      # 3 个子 Agent 提示词
├── workflows/literature_review.json  # 综述 DAG 蓝图
├── templates/                   # dashboard / search / review 三页
├── static/js/veroscholar.js     # 前端逻辑（VS 命名空间）
├── static/css/veroscholar.css   # design-system 变量风格
├── i18n/en.yml + zh-CN.yml      # 英文键双语映射
└── tests/                       # 单元测试（105 用例）
```

## 安装与启用

通过系统插件管理页面上传/启用本插件：

1. `python_dependencies.required` 声明 `feedparser`（arXiv 适配器运行时惰性加载）；
2. 启用时自动执行 `migrations/*.sql`（幂等，记录 `schema_version`）；
3. 卸载时 `DROP SCHEMA veroscholar CASCADE` 零残留。

## 配置项

在插件设置页可配置：

| 配置 | 默认 | 说明 |
|------|------|------|
| `semantic_scholar_api_key` | — | 可选，提高语义检索速率限制 |
| `openalex_api_key` | — | 可选，提高 OpenAlex 请求限额 |
| `default_search_limit` | 30 | 默认检索条数（5–100） |
| `arxiv_enabled` | true | arXiv 数据源开关 |
| `semantic_enabled` | true | Semantic Scholar 数据源开关 |
| `openalex_enabled` | true | OpenAlex 数据源开关 |
| `zh_enabled` | true | 中文文献源开关（OpenAlex 中文作品） |

## 测试

```bash
cd F:\Sites\VeroRun
python -m unittest plugins.veroscholar.tests.test_adapters \
                       plugins.veroscholar.tests.test_models \
                       plugins.veroscholar.tests.test_routes \
                       plugins.veroscholar.tests.test_services \
                       plugins.veroscholar.tests.test_workflow \
                       plugins.veroscholar.tests.test_zh_adapter \
                       plugins.veroscholar.tests.test_fulltext -v
```

116 用例覆盖：适配器字段映射 / 摘要重建 / 错误传播、中文源 language:zh 过滤与无 DOI 保留、数据层 SQL 构造、路由鉴权（401/302/静态豁免）、页面与 API 端点、引用校验权威验真五态、综述去重节点、检索过滤、预印本导出、论文问答与翻译、标签系统与阅读状态、全文分块与问答注入降级、综述回写 reviews 表、全文删除路由（成功/论文不存在/非法 file_id）、多源检索 per_source 配额。

## 依赖

- 运行时：`feedparser` + `pypdf`（Python 依赖）、PostgreSQL + `pgvector`（可选，embedding 列）
- 环境：插件标准 v1.5+，`min_app_version: 0.58.0`
