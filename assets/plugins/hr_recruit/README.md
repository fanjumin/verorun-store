# HR Recruit（hr_recruit）

招聘智能协办插件 —— **AI 只建议，人做决定；决策全程留痕**。

> 版本：v1.1.0 · 依据：plugin-standard-v1.7 / i18n-standard v1.0 / VeroRun v0.61.0
> 状态：完整实现骨架 + 文档，尚未部署到 VeroRun 仓库。

---

## 1. 定位与边界

### 1.1 定位

面向**已在自有服务器部署 VeroRun 的中大型企业**，提供可插拔的招聘能力层：
把 JD 撰写、简历结构化、人才库检索、面试协调这些高重复度环节交给 Agent，
同时把「谁被推进、谁被淘汰」的决定权与完整审计链留在人手里。

### 1.2 明确不做

| 不做 | 原因 |
|------|------|
| 自动淘汰结论 | PIPL 第 24 条 / GDPR Art.22 / 欧盟 AI Act 均要求人工实质参与 |
| 情绪识别、人格推断 | 欧盟 AI Act Annex III 明令禁止职场情绪分析 |
| AI 视频面试自动评分 | 高风险，需额外合规评估（如 Illinois AIVIA 书面同意） |
| 替代 ATS | 首版定位为 ATS 旁挂的增强层 |

### 1.3 差异化

不在「谁筛得更准」上与北森 / Moka 竞争，而在**数据主权 + 决策可追溯**上竞争：

| 维度 | 北森 / Moka | hr_recruit |
|------|-------------|------------|
| 形态 | 全栈 ATS SaaS（闭源） | 客户自有服务器上的可插拔能力层 |
| 数据位置 | 厂商云 | **客户服务器，可完全不出内网** |
| 模型 | 厂商自研垂直模型 | UnifiedLLM 网关，可切 `tier=local` |
| 可审计性 | 厂商报表 | 决策日志 + prompt 版本 + 证据引用，可全量导出 |
| 扩展性 | 厂商排期 | 插件 + DAG + 事件总线，客户自编排 |

---

## 2. 能力矩阵与上线顺序

| 环节 | 风险 | 自动化形态 | 人工介入 | 建议顺序 |
|------|------|-----------|---------|---------|
| 面试排期协调 | 低 | 全自动（跨日历求交集） | 无 | **① 首个上线** |
| JD 生成 + 偏见审查 | 低 | 四阶段协议生成 | 发布前人工签字 | ② |
| 候选人状态通知 | 低 | 全自动 | 无 | ③ |
| 简历解析与结构化 | 中 | AI 抽取 + 置信度标注 | 低置信度人工校正 | ④ |
| 人才库语义检索 | 中 | RAG + 证据引用 | 结果为候选集，非结论 | ⑤ |
| 保留期销毁 | 中 | Cron + 密钥销毁 | 执行前清单复核 | ⑥ |
| 匹配评分排序 | 中 | v0.2 | **必须人工复核** | v0.2 |
| 自动淘汰 / 情绪分析 | 高/禁止 | **不做** | — | — |

**上线顺序必须与风险顺序一致**：先做失败代价仅一次改期的排期，建立团队信任；
评分排序类能力留到最后。

**首版范围（已拍板）：v0.1 六项能力全量上线**，即上表 ①-⑥：
`hr.jd_draft` / `hr.bias_review` / `hr.resume_parse` / `hr.talent_search` /
`hr.interview_schedule` / `hr.retention_purge`（与 `plugin.json` capabilities 一一对应）。
第 ③ 行「候选人状态通知」为流程细节能力，由 DAG `wait` + 事件 `hr.application.stage_changed`
承载，不单列 capabilities。「匹配评分排序」明确推后至 v0.2。

---

## 3. 目录结构

```
plugins/hr_recruit/
├── plugin.json                  # manifest：能力/菜单/设置/权限/钩子
├── __init__.py                  # HRRecruitPlugin(BasePlugin)
├── models.py                    # 连接工厂（借池 + search_path + 归还）
├── crypto.py                    # 信封加密：DEK/KEK 与密钥销毁
├── llm.py                       # UnifiedLLM 封装（tier + prompt 版本）
├── jd.py                        # JD 生成四阶段 + 人工签字
├── bias.py                      # 双通道偏见审查 + impact ratio
├── talent_search.py             # 人才库检索（向量 / 关键词降级）
├── scheduling.py                # 排期与阶段通知
├── retention.py                 # 保留期扫描与销毁
├── dag_nodes.py                 # DAG 节点处理器集合
├── hr_skill.py                  # 自包含技能（供对话/工作流调用）
├── routes.py                    # Blueprint /admin/hr-recruit/*
├── parsers/
│   └── resume_parser.py         # 纯 Python 简历解析（禁外部二进制）
├── tools/
│   └── mcp_server.py            # MCP 工具服务器（stdio JSON-RPC）
├── migrations/
│   └── v1.0.0_init.sql          # hr_recruit schema DDL
├── templates/
│   ├── hr_recruit.html          # 内联 partial（首行 JS，禁 script 标签）
│   └── hr_recruit_iframe.html   # iframe 完整页（含主题同步）
├── i18n/
│   ├── en.yml                   # identity 映射（强制）
│   └── zh-CN.yml                # 中文映射（强制）
├── agents/
│   └── hr_recruiter_prompt.md
├── agent_matrix_additions/      # 遗留参考（v1.0.0 起不再复制，仅存档；见 §4.1）
│   ├── 22-hr_recruiter.yaml
│   └── prompts/hr_recruiter.md
├── tests/                       # 单元测试与回归说明
├── docs/                        # 合规、验收、部署文档
├── SKILL.md / USAGE.md / README.md / CHANGELOG.md
└── requirements.txt / setup.sh
```

---

## 4. 安装与配置

### 4.1 部署步骤

```bash
# 1) 复制插件目录到仓库
cp -r hr_recruit_plugin <repo>/plugins/hr_recruit

# 2) 安装依赖与自检
bash <repo>/plugins/hr_recruit/setup.sh

# 3) 重启 admin 服务
#    （能力经 plugin.json 的 agent_role=office 由平台启动时自动聚合，
#      无需人工复制任何角色 YAML）
```

### 4.2 必填配置

| 设置项 | 说明 |
|--------|------|
| `hr_master_key`（KEK，**必填**） | 包裹每位候选人 DEK 的主密钥。**未配置将拒绝启用** |
| `llm_tier` | `high`/`standard`/`cheap`/`local`；`local` 保证简历不出内网 |
| `resume_raw_days` | 原始简历保留期，默认 180 天 |
| `candidate_pii_days` | 身份字段保留期，默认 365 天 |
| `max_upload_mb` | 简历上传字节上限，默认 10 MB（R-11 上传加固） |
| `purge_mode` | `crypto_shred`（默认，销毁密钥+脱敏）/ `hard_delete`（连带删投递链与主档） |
| `notify_before_days` | 销毁前提前通知天数，默认 7 天 |
| `bias_scan_rate_limit` | 偏见审查限流（次/分钟），默认 30 |
| `resume_upload_rate_limit` | 简历上传限流（次/分钟），默认 10 |

> **决策日志永久保留、只追加**（R-8 决策）：`hr_decision_log` 仅存脱敏 `ref_hash`，
> 不按天数删除，自动满足 AI Act ≥180 天日志留存要求。
>
> **KEK 丢失 = 所有候选人 PII 永久不可读。** 请在启用前完成备份与交接。

---

## 5. 数据模型（`hr_recruit` schema）

| 表 | 用途 |
|----|------|
| `hr_job` / `hr_job_description` | 职位与 JD 版本链（含 Reviewer 问题、人工签字人） |
| `hr_candidate` | 候选人主档（PII 字段 DEK 加密） |
| `hr_candidate_dek` | 数据密钥；**销毁此行 = 所有副本不可解** |
| `hr_resume_raw` | 原始简历与抽取文本（加密，最短保留期） |
| `hr_candidate_profile` | 结构化档案（非 PII，明文以便检索统计） |
| `hr_talent_embedding` | 人才库向量（维度运行时校正，禁止硬编码） |
| `hr_application` | 投递/流程实例（`source` 渠道：resume_upload/talent_search/manual；同职同候选人仅一条 active 申请） |
| **`hr_decision_log`** | **合规核心：决策日志，PG RULE 禁 UPDATE/DELETE** |
| `hr_bias_metric` | impact ratio 统计（NYC LL144） |
| `hr_purge_receipt` | 销毁回执（审计凭证） |

---

## 6. 保留期销毁（crypto-shredding）

四层残留中，前三层（主库 / 文件 / 向量）都能精确定位删除；
**第四层（vault 备份与快照）无法逐份定位** —— 这是多数 HR 系统失守之处。

本插件的解法是**不追赶副本，而是让副本全部不可解**：

```
写入：每位候选人独立 DEK → PII 用 DEK 加密落库 → DEK 由 KEK 包裹存入 hr_candidate_dek
销毁：删除 DEK 行 → 主库 / 向量副本 / vault 备份 / 增量快照中的密文同时永久不可解
```

执行顺序铁律：**先销毁密钥，再删数据**（反之中途失败会留下明文残留）。

**密钥派生与密文版本（v2，第四轮 B12）**

| 项 | v2（新数据，默认写入） | v1（存量，仅兼容读取） |
|----|------------------------|------------------------|
| KEK | `PBKDF2-HMAC-SHA256(主密钥, salt, 200000)` 派生 32B | 主密钥补零/截断至 32B |
| 密文格式 | `0x02 ‖ nonce ‖ ciphertext` | `nonce ‖ ciphertext` |
| AAD 绑定 | 候选人 ID + 用途（字段级）/ 候选人 ID（DEK 级） | 无 |
| `hr_candidate_dek.kek_id` | `hr_recruit_kek_pbkdf2_v2` | `hr_recruit_default_kek` |

升级**不做数据迁移、不做重加密**：解密/解包一律先试 v2，失败再回落 v1，
历史密文继续可解（选项 A 版本化回退）。PBKDF2 派生结果按主密钥材料在进程内缓存，
每个 worker 只付一次派生代价。

> **与 vault 的关系**：销毁不改写历史备份。已销毁候选人的 PII 在历史备份中仍以密文存在，
> 但 DEK 已销毁而永久不可解。从 vault 恢复备份**不会**导致已销毁 PII 复活。

---

## 7. API

所有接口位于 `/admin/hr-recruit/api/*`，统一返回 `{"success": bool, ...}`。

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/stats` | 仪表盘统计 |
| GET/POST | `/api/jobs` | 职位列表 / 创建 |
| POST | `/api/jobs/<id>/jd` | 生成 JD + 偏见审查 |
| GET | `/api/jobs/<id>/jd/versions` | JD 版本链 |
| POST | `/api/jobs/<id>/jd/<v>/signoff` | **人工签字放行**（override 必须附理由，独立 `jd_publish_override` 留痕） |
| POST | `/api/bias/scan` | 偏见审查（限流；豁免一律按文本语义语境判定，不接受 `job_category` 参数） |
| POST | `/api/resume/parse` | 简历上传解析（限流；受 `max_upload_mb` 字节上限与页数上限约束，需 `consent`） |
| POST | `/api/talent/search` | 人才库检索 |
| POST | `/api/schedule/propose` | 排期提案（前端表单：面试官忙时段 / 节假日 / 时长 / 窗口，纯计算不写库） |
| GET | `/api/applications` | 投递管道元数据（无 PII） |
| POST | `/api/applications` | 创建投递记录（校验职位 open / 候选人存在 / 同职防重复，写入 evidence 三要素起点留痕） |
| POST | `/api/applications/<id>/stage` | 推进阶段（禁止跳级/回退；终态 hired/rejected 自动 closed 并留痕） |
| GET | `/api/retention/stats` · `/scan` · `/receipts` | 保留期统计 / 扫描 / 凭证 |
| POST | `/api/retention/purge` | 手动销毁。单候选人：须键入其 `ref_hash` 回填（服务端与库内记录逐字比对，不符即拒）；批量：须 `confirm: true` 且一次不超过 `purge_batch_max`（默认 50） |
| GET | `/api/candidates` · `/api/candidates/<id>` · `/api/candidates/<id>/resume` | 候选人列表 / 详情 / 简历（详情解密每次留痕 view） |
| GET | `/api/decisions` | 决策日志（脱敏） |

## 8. MCP 工具

`plugin.json` 注册 `mcp_servers[0].name = "hr"`，平台暴露为 `mcp__hr_recruit__hr__*`：

| 工具 | 能力 | 说明 |
|------|------|------|
| `hr_job_list` | 职位 | 在招职位列表 |
| `hr_jd_versions` | JD | 某职位 JD 版本链（版本/模型/签字状态/摘要），只读 |
| `hr_bias_scan` | 偏见审查 | 歧视性表述规则扫描（本地快路径，不调 LLM） |
| `hr_talent_search` | 人才检索 | 语义检索（脱敏候选集） |
| `hr_schedule_propose` | 面试排期 | 跨日历求空闲时段（纯计算，不写库） |
| `hr_retention_stats` | 保留期销毁 | 统计（到期/即将到期/已销毁） |
| `hr_decision_log` | 决策留痕 | 最近决策日志（脱敏，仅含 `candidate_ref`） |

**暴露面边界（六项能力全量上线的 v0.1）：**
MCP 层**只暴露快路径与只读/统计工具**。内核 `McpClient` 请求默认超时 15s 且无 per-plugin 配置，
故 **LLM 长时调用（JD 生成 / 简历解析）与不可逆动作（手动销毁）不经过 MCP**，改走 HTTP API 或 `hr_skill`
（前者受 `admin:access` 保护，后者在宿主 Agent 内同步执行并受策略约束）。

**合规铁律：MCP 工具绝不返回候选人 PII 明文。**

## 9. DAG 节点

`hr.jd_draft` · `hr.bias_review` · `hr.resume_parse` · `hr.talent_search` · `hr.schedule` · `hr.purge`

> ⚠ 内核 `approval` / `sub_workflow` / `script` 三个节点目前为占位实现。
> v0.1 用「人工签字 API + 决策日志」替代录用审批，待内核补齐后再切换。

---

## 10. 相关文档

- `docs/00-index.md` —— 文档索引与部署清单
- `docs/compliance.md` —— 合规要求矩阵与自检表
- `docs/acceptance.md` —— 功能与合规验收清单
- `USAGE.md` —— 使用说明
- `SKILL.md` —— 技能说明
- `tests/REGRESSION_TESTING.md` —— 回归测试说明
