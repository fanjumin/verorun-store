# VeroRun Risk Control Plugin - 统一风控决策引擎

## 概述

VeroRun Risk Control 是一套企业级统一风控决策引擎插件，覆盖六大风险场景：

- **账户与身份**：登录/注册/短信验证风控
- **资金与计费**：支付/充值/退款风控
- **AI 资源与成本**：AI 调用配额/预算熔断
- **插件供应链**：插件安装/更新审计
- **内容与合规**：内容发布审核
- **商城交易**：下单/优惠券防薅

## 核心特性

- **毫秒级决策**：同步决策 P99 < 100ms，超时自动降级
- **规则引擎**：自研 DSL + AST 白名单求值，支持灰度/版本/回放
- **名单库**：黑/灰/白名单 × 设备/IP/号码/账户/卡
- **统一限流**：固定窗口/滑动窗口/配额型限流
- **可解释可追溯**：每条决策留痕 ≥5 年，可回答"为什么拒"
- **AI 增强**：经系统统一网关调用 LLM 语义审核（不注册自有 Agent）

## 技术架构

```
接入层 → 风控决策层（plugins/risk_control）→ 数据层（schema risk）
```

## 安装

1. 将插件目录放置于 `plugins/risk_control/`
2. 在管理后台启用插件
3. 系统自动创建 risk schema 和表结构

## 使用方式

### 业务方接入

```python
_pm = flask.current_app.extensions.get('plugin_manager')
_rc = _pm.get_instance('risk_control') if (_pm and _pm.is_enabled('risk_control')) else None

if _rc:
    decision = _rc.decide('login', {
        'user_id': uid,
        'device_id': fp,
        'ip': ip,
        'payload': {'method': 'sms', 'fails_30d': n},
    })
```

### 决策 API

```
POST /admin/risk/api/v1/decision
{
    "scene": "login",
    "user_id": "u_1001",
    "device_id": "fp_9f2a...",
    "ip": "1.2.3.4",
    "payload": {"amount_fen": 6800}
}
```

## 配置项

| 配置 | 默认值 | 说明 |
|------|--------|------|
| sync_timeout_ms | 100 | 同步决策超时(ms) |
| fallback_decision | pass | 降级动作 |
| captcha_threshold | 5 | 登录失败触发验证码次数 |
| refund_auto_threshold_fen | 10000 | 小额自动退款阈值(分) |
| ai_budget_daily_fen | 0 | AI 单日预算上限(分, 0=不限制) |

## 合规认证

- ✅ 无 eval/exec/subprocess/pickle
- ✅ 全部参数化查询
- ✅ 零硬编码凭据
- ✅ 连接池规范（借还成对）
- ✅ i18n 国际化
- ✅ 统一网关绑定（不建 Agent）

## 设计与取舍说明

> 本节记录风控插件在真实系统集成中的关键设计与取舍（P2 阶段归档，2026-08-28）。

### 名单库优先级语义

名单检查为**单一决策**，不叠加计分：命中即按 `black > gray > white` 优先级返回该条命中，
不会将多条命中组合成累计风险分。

- `black` → `reject`（score=100）
- `gray` → `challenge`（score=60，走人工审核/验证码）
- `white` → 返回 None：**仅豁免名单检查**，仍受频控与规则引擎约束（不豁免一切风控）
- 多维度（user/device/ip/agent）合并为一次查询，返回跨维度最高优先级有效命中；
  过期记录在 SQL 层排除，保证与原逐维度实现语义等价。

### 编排模型（P2-7）

插件**无自建编排、无直接 EventBus 订阅**：

- 事件订阅唯一通道为 `get_event_handlers()`（由插件管理器统一注册）
- 定时任务唯一通道为 `register_jobs()`（scheduler.py 声明）
- 旧的生命周期钩子 `on_enable/on_disable` 已被移除（管理器仅走 `setup()/activate()/deactivate()`）

### 决策 API 鉴权（P2-6）

`POST /admin/risk/api/v1/decision` 仅限管理员（后台 SSO）调用，业务侧请使用
插件实例方法 `decide()` 经主站进程内调用，避免对外暴露决策接口。

### 名单脱敏取舍（R-5）

名单库以"维度+值"明文存储，未做值脱敏。原因：名单由管理员经后台手工维护，
数据规模小且无对外查询接口；脱敏会破坏精确匹配能力。若未来开放对外查询，
须先落地脱敏索引再开放。

### 频控 PG 兜底失效模式（N-2）

Redis 不可用时的 PG 原子限流兜底路径中，若**连接获取失败**，异常会上抛至
`make_decision` 外层 catch，最终返回**场景 fallback**（多数场景为 pass）；
原实现由 `check_pg_window_atomic` 内部捕获后返回限流 **reject**（score 80）。

即 PG 故障期间限流兜底的失效模式从「硬拦截」变为「场景降级」，二者皆可接受，
运维巡检时请注意：PG 兜底失效不会产生误拦截，但会短暂放大流量，需结合场景
fallback 配置（`fallback_decision`）统筹观察。

## 版本

- v1.0.0 - 初始版本
