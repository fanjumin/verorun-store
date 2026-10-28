# Survey & Forms（survey 插件 v2.0）

VeroRun 原生**市场调研问卷**系统。v2.0 起心理测评能力已拆到独立插件
`plugins.assessment`，两者共用一个非插件形态的内核 `plugins._survey_core`。

业务语义 100% 收敛在本插件内：独立 `survey` schema、独立蓝图、独立静态资产，
内核仅提供插件框架、连接池与 i18n 播种能力。

## 依赖与边界

```
survey ─┐
        ├──> plugins._survey_core      二者互不 import
assessment ─┘
```

- `_survey_core` 无 `plugin.json`、前缀 `_` 被 `discovery.py` 跳过，不出现在插件中心，
  随代码分发；用户视角只有 `survey` 与 `assessment` 两个条目。
- 本插件的服务层（`services/*.py`）是对内核的**转发/薄适配**，不再持有问卷领域实现。
  唯一的自有模块是 `stats.py`（市场调研统计聚合）。
- 计分、常模、分级报告、敏感信息二次确认与审计随测评一起移出。历史行里的
  `scoring_json` / `score_json` / `result_json` 列与 `survey_norms` /
  `survey_audit_log` 表**保持原样不删数据**，但不再参与任何读写路径。

## 功能面

- **问卷设计器**：11 类题型（短文本/长文本/单选/多选/下拉/量表/星级/数字/日期/
  排序/分段标题，其中前 10 类可作答、分段标题为结构元素）、必答、选项、量表区间、
  分页题块、JSON 导入。
- **作答链路**：三段式状态机（start → submit → completed），匿名分享 token 与
  登录态双通道；配额、起止时间、访问密码、IP/Token 频控、蜜罐字段、幂等提交。
- **分页作答**：题目带 `page`（≥1，1..N 连续）即分页，服务端同时下发 `pages`
  目录；请求体仍允许一次提交全卷，分页纯属交互层。未提交前答案只留在浏览器
  草稿（localStorage），回退改答不丢已填值。
- **随机化**：题目可按页内/全卷乱序，选项可轮转/打乱，`anchors` 中的选项
  （"其他""以上都不是"）固定末尾。顺序由 `sha256(response_id:survey_id)` 派生、
  **不落库**，同一答卷每次进入顺序一致。
- **条件逻辑**：题目可配 `show_if`（按前序答案显隐，支持 `all`/`any`/`not` 一层
  嵌套）与 `skip_to`（答某值跳转到后续题），另有 `piping` 文本引用（`{q1.label}`）。
  规则随问卷版本存于 `schema_json`，不单独建表；**服务端在提交时按同一套求值器
  二次判定**，隐藏题既不校验必答也不落库，响应体回报 `hidden_by_logic` 计数。
- **答卷管理**：admin 答卷列表/明细、CSV 导出（单并发锁）、保留期匿名化 sweeper。
- **统计看板**：完成率、30 日趋势、题型分布；5 分钟快照 + 实时聚合双口径。
- **本人回看**：`/survey/<slug>/result/<id>` 渲染作答明细页（页面，不是 JSON 端点）。
- **F24 网站嵌入**：`?embed=1` 无 chrome iframe 视图；`embed.js` 挂载点渲染器
  （Shadow DOM 隔离、探活失败自隐藏）；公开 API 白名单脱敏。

## 蓝图与端点

| 蓝图 | 前缀 | 鉴权 | 说明 |
|---|---|---|---|
| `admin_bp` | `/plugin/survey` | admin 会话 | 设计器/答卷/统计 API 与 3 个 iframe 页面 |
| `web_bp` | `/survey` | 登录态或匿名 token | 作答页、start/submit、本人作答回看页 |
| `mp_bp` | `/api/v1/mini-program/survey` | 小程序 JWT，禁匿名 | 小程序五端点 |
| `public_bp` | `/api/v1/survey/public` | 匿名（同域） | F24 公开 JSON 契约：`<slug>` / `<slug>/start` / `<slug>/submit` |

分页与随机化没有独立蓝图，挂在既有端点上：

| 端点 | 说明 |
|---|---|
| `GET /survey/<slug>/schema?response_id=&token=` | 带上归属正确的 `response_id` 才用该答卷的随机种子；归属不符或无参一律回退卷级种子 |
| `GET/PUT /plugin/survey/api/surveys/<id>/randomization` | 读写随机化配置；`closed` 后拒绝（409），`active` 时可改（答案按 qid 存，顺序不落库，改配置不破坏已提交数据） |

公开端统一门禁：仅 `active` + 允许匿名 + 无访问密码 + 未声明敏感策略（内核
`ScoringPolicy.forbids_public_embed`）的问卷可嵌；其余一律统一 404，不暴露问卷
为何不可嵌。公开载荷经 `services/public.py` 白名单投影：题级评分映射、dimension、
reverse、白名单与密码哈希永不下发；`logic` 的 `show_if`/`skip_to` 与 `piping`
必须下发（前端要按答案实时显隐），它们只表达"哪些题依赖哪些答案"，不含计分语义。

## 条件逻辑（DSL v1）

规则写在题目上，随问卷版本存进 `schema_json`，不建表：

```json
{
  "qid": "q3",
  "logic": {
    "show_if": {"all": [
      {"qid": "q1", "op": "eq", "value": "yes"},
      {"qid": "q2", "op": "gte", "value": 18}
    ]},
    "skip_to": {"if": {"qid": "q3", "op": "in", "value": ["none"]}, "target": "q7"}
  },
  "piping": {"text": "您选择了 {q1.label}，请说明原因"}
}
```

- **算子**：`eq ne gt gte lt lte in not_in contains answered unanswered`；
- **组合子**：`all` / `any` / `not`，最多一层嵌套（更深的按 `logic_too_deep` 拒绝）；
- **未作答语义**：题未作答时一切比较为假（含 `ne`/`not_in`），只有 `unanswered`
  为真——否则一张空表会把所有依赖题默认显示出来；
- **隐藏连带**：被隐藏的题视作未作答，下游依赖它的条件题跟着消失（不留幽灵答案）；
- **设计期校验**（`validate_schema`，保存与 F18 导入同路径）：引用不存在的题
  `logic_unknown_qid`、自引用 `logic_self_reference`、跳转目标不在本题之后
  `skip_target_not_forward`、依赖环 `logic_cycle`、管道引用不存在的题
  `piping_unknown_qid`、**标题写了 `{qid.label}` 却没配 `piping.text`
  `piping_not_configured`**；
- **提交期强制**：`services/engine.complete_answer` 固定注入
  `logic.SurveyLogic`，隐藏题不校验必答、不落库，响应体回报 `hidden_by_logic`；
- **管道替换只发生在展示层**，不写回答案——历史答卷的 `value_text` 已存当时的
  选项文本，改题不会污染原答。

关于 `piping` 有一个容易踩的语义：**被替换的模板是题目标题，`piping.text`
只是"开启替换"的开关**（运行期 `pipingText()` / `titleOf()` 只看它在不在，
开关内容本身不渲染）。所以正确用法是把 `{q1.label}` 写进标题、再给该题配上
任意非空 `piping.text`；只写标题不配开关，占位符会原样露给受访者，保存时直接被
`piping_not_configured` 拦下。标题里引用了不存在的题同样设计期拒绝——否则会
渲染成空串，留下"你选了 "这种半句话。

前端求值器是 `static/js/logic.js`（与内核 `logic.py` 同一套语义，8 组用例跨语言
对齐）；`embed.js` 在宿主页动态加载它，加载失败则降级为"全部可见"，写入仍由
服务端兜底。

## Agent（MCP 只读面）

`tools/mcp_server.py`，stdio 行分隔 JSON-RPC 2.0，四工具全部只读：

| 工具 | 约束 |
|---|---|
| `list_surveys` | status 过滤，limit ≤ 200 |
| `get_survey_schema` | 复用公开白名单投影 |
| `get_answers` | **`cursor` 与 `limit` 必填**，keyset 分页（首页 cursor=0），limit ≤ 200；无 cursor 返回分页说明并拒绝 |
| `get_stats` | 优先读 `overview` 快照（`source=snapshot`），无快照实时群体聚合（`source=live`） |

建卷、改卷、导出、删除等写操作**不暴露**给 MCP。Agent 可见性由内核
`allowed_tools` 白名单控制（`mcp__survey__survey__*`）。

## 数据库

迁移位于 `migrations/`，启用/升级时由 BasePlugin 标准迁移器执行（台账 + advisory
lock + 事务，幂等）：

- `v1.0.0_init.sql`：survey_defs / survey_questions / survey_answers /
  survey_answer_items / survey_stats_cache / survey_rate_events /
  survey_notify_log 七表与索引。
- `v1.1.0_assessment.sql`：**历史迁移，v2.0 起不再写入**。它引入的
  `scoring_json` / `score_json` / `result_json` 三列与 `survey_norms` /
  `survey_audit_log` 两表保留在库中（不 DROP，避免删历史数据），但没有代码路径
  再读或写它们。继续做测评请用 `assessment` 插件（独立 `assessment` schema）。
- `v1.2.0_pages.sql`：`survey_questions.page`（默认 1，与 `schema_json` 双写）
  与 `survey_randomization`（一问卷一行：`question_order` / `option_order` /
  `anchors` / `seed_source`）。

条件逻辑**没有**新增列或表：`logic` 随题目双写进 `survey_questions.logic_json` 与
`schema_json`（既有双写约定），`piping` 只在 `schema_json` 里。运行期求值一律读
**冻结快照**（`engine.frozen_questions`），改表或改题目草稿都不会影响已发布问卷
的规则。

全部表位于独立 `survey` schema，连接经共享池并 `SET search_path TO survey, public`。
卸载仍走 `DROP SCHEMA survey CASCADE`——因为测评数据已不在这个 schema 内，
卸载市场调研插件不会再连带删掉测评数据，这正是本次拆分要解决的问题。

## 配置与保留期

两类配置**层级不同**，读法也不同，混为一谈会导致"改了不生效"：

| 键 | 层级 | 默认 | 真实读取处 | 可改入口 |
|---|---|---|---|---|
| `stats_snapshot_minutes` | 插件级 `config` | 5 | `__init__.py:95` | 仅 API |
| `default_answer_ttl_days` | 插件级 `config` | 180 | `scheduler.py` | 仅 API |
| `sweeper_batch_size` | 插件级 `config` | 500 | `scheduler.py` | 仅 API |
| `in_progress_ttl_days` | 插件级 `config` | 7 | `scheduler.py` | 仅 API（未列入 `settings_schema`） |
| `allow_anonymous` | 问卷级 `settings_json` | true | `_survey_core/engine.py`、`_survey_core/public.py` | 设计器 ✅ |
| `quota` | 问卷级 | 0（不限） | `_survey_core/engine.py` | 设计器 ✅ |
| `whitelist` / `password_hash` | 问卷级 | — | `_survey_core/engine.py` | 设计器 ✅ |
| `public_base_url` / `privacy_notice` | 问卷级 | 请求 host / 空 | `routes.py`、`_survey_core/public.py` | 设计器 ✅ |
| `submit_rate_limit` | 问卷级 | 30 | `routes.py`（3 处调用点） | ❌ 暂无输入口，实际取默认 |
| `start_rate_limit` | 问卷级 | 20 | `routes.py`（3 处调用点） | ❌ 暂无输入口，实际取默认 |
| `max_questions_per_survey` | 问卷级 | 100 | `routes.py`（PUT 保存） | ❌ 暂无输入口（`config` 里的同名项不被读） |
| `public_link_ip_hash_guard` | 问卷级 | false | `routes.py` | ❌ 暂无输入口（同上） |

保留期为 0 表示永久（仅建议内部运维问卷）。sweeper 每日 03:15 执行，幂等。导出并发固定为
每进程 1（`_survey_core/exporter.py` 模块级信号量），不是可配项。

`public_link_ip_hash_guard` 的语义在拆分时被拆成两件事（旧版混在 `is_assessment`
一个布尔里）：**算不算**每日 IP 哈希由内核 `ScoringPolicy.record_ip_hash` 决定
（市场调研 true，测评 false）；**落不落库**由本开关决定。false（默认）时哈希只进
频控事件表，不写入答卷行。

**改插件级配置后怎么生效**：`settings_schema` 目前没有后台表单，只能经
`POST /admin/plugins/survey/config` 写入注册表；运行实例的 `_config` 只在插件实例化时注入
一次，因此**必须重启服务或重新启用插件**才生效。标 ❌ 的键是运营参数但尚无配置
入口，属已知待接线项。

## 合规边界

- 真匿名：匿名链接作答 `user_id` 落 NULL；登录态 + 匿名标记组合被禁止。
- 公开嵌入、MCP 以及作答端 web/小程序 schema 端点均为白名单投影，题级评分映射、
  dimension、reverse 标记不下发，评分资产不外泄。
- 敏感信息（测评结果、专项同意、明细审计）不属本插件职责，见 `plugins/assessment`。
- 站点嵌入与 site_builder 的正式 `survey_form` 控件（Tier 3）属受保护插件审批批；
  当前 Tier 2 通过基础 `embed` / `code_block` 控件粘贴片段投放。

## 测试

纯函数离线测试（零三方依赖，无需数据库）：

```bash
python -m unittest discover -s plugins/survey/tests -p 'test_*.py' -v
```

覆盖：schema/答案校验、F24 脱敏白名单（含历史测评快照的剥离）、引擎纯函数
（频控键、IP 哈希、蜜罐、密码哈希）、SQL 契约（保存不再触碰 `scoring_json`）、
导出公式注入防护与统计构造器。内核侧另有 `test_pages.py` 锁住分页归一化、页目录
投影与随机化可复现性（同种子幂等、页归属不变、anchors 固定末尾），
`test_logic.py` 锁住条件 DSL 的设计期校验（环/自引用/跳转方向）、求值语义
（未作答、多选 in/contains、隐藏连带、历史快照 fail-open）与提交期剔除。
HTTP/SQL 层行为（频控 429、MCP 冷启动、`allowed_tools` 可见性、嵌入实答、
分页翻页与改答显隐）属部署环境验证项。
