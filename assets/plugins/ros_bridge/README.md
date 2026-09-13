# ROS Bridge

Robot OS bridge gateway（机器人桥接网关）：通过 rosbridge（WebSocket）连接
ROS 2 舰队，提供机器人注册与鉴权、连接监管、图发现；后续阶段将接入遥测入库、
白名单指令、任务管理与进化飞轮推送。

> 插件自有数据独立存放于 PostgreSQL schema `ros_bridge`，卸载即零残留删除。

## 当前状态（P1 骨架 · P2 遥测 · P3 指令与审批 · P4 Agent 指挥）

- 迁移：`migrations/v0.1.0_init.sql`（robots / ros_telemetry / missions / command_audit），
  采用 advisory lock + schema_migrations run-once（多 worker 安全），键 `0x524F53425247`
- 机器人 CRUD 与密钥管理：`/admin/ros/robots*`（JWT `is_admin` 守卫）
- 连接监管：`BridgeRuntime` 自管线程，退避重连（1s→配置上限）、心跳探活
  （`/rosapi/get_topics`）、在线/离线事件 `ros.robot.online/offline`
- 图发现：`GET /admin/ros/robots/<id>/graph`（即时 rosapi 查询）
- 遥测（P2）：机器人 `metadata.telemetry_topics` 配置话题 → 连接/重连后订阅
  （`queue_length=1` 保最新 + 协议级 `telemetry_throttle_ms`）→ 客户端聚合整形
  （每话题 ≥ `telemetry_store_min_interval_sec` 入库一行）→ `ros_telemetry`；
  查询 `GET /admin/ros/robots/<id>/telemetry`（不带 `metric` 返回最近列表，
  带 `metric` 返回点路径时间序列，供 ECharts）
- 保留清理 job（每日 03:40 按 `telemetry_retention_days`）
- 管理页：菜单 `Tools / ROS 桥接`（`window.l_ros_admin`，含机器人/图/遥测/指令/审批/任务面板）

P4 Agent 指挥（MCP 通道）：

- `plugin.json` 声明 `mcp_servers`（name=`ros`）→ 平台将工具暴露为 `mcp__ros_bridge__ros__*`
- 工具 `tools/mcp_server.py`：读类（`list_robots` / `robot_status` / `telemetry_query` /
  `list_missions`）走 DB 快路径；写类（`send_command` / `cancel_mission`）只做
  CommandGate 裁决 + 落库
- `send_command` 语义：denied → 审计；requires_approval → 进 P3 人工审批流；
  allowed 且在线 → audit(allowed) + mission `status='queued'`，由 resident runtime
  每 ~2s sweep 认领（`queued → goal_sent` 原子转换）并在应用进程内真分发
  （闸3：离线永不下发；白名单/钩子链在认领时二次校验）
- missions 状态取值：`queued`（P4 新增，零 DDL）`goal_sent` `active`
  `succeeded` `failed` `cancelled` `expired`
- `min_app_version 0.61.0`：能力聚合到 `ops` 核心角色 + MCP 装载的版本门槛

## 启用要求

- 运行环境 Python ≥ 3.8 且已安装 `websockets>=12`（见根 `requirements.txt`；
  服务器 venv 需另行 `pip install "websockets>=12"`）
- PostgreSQL 13+（`gen_random_uuid()` 内建，无扩展依赖）
- 管理端（Admin :8084）运行中，插件管理启用 ros_bridge

## 环境变量

使用系统统一 PG 配置：`PG_HOST / PG_PORT / PG_DB / PG_USER / PG_PASSWORD`
（与其它插件一致，经 `plugins._base.db` 借池）。

## 钩子与事件

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `cogevolution.owner_allowed` | filter（提供） | `owner_type='robot'` 时按 `robots.enabled` 仲裁；其余域透传 |
| `ros.robot.online / offline` | event（emit） | 连接状态翻转，负载 `robot_id`/`robot_key` |
| `ros.mission.completed` | event（emit） | 任务终态，负载 `robot_key`/`mission_id`/`status` |
| `cogevolution.curation.submit` | event（emit，P3b） | mission 终态策展推送（受 `curation_push_enabled` 门控） |
| `cogevolution.improvement.outcome` | event（emit，P3b） | 每日 applied ros 提案绩效打点 |

## 说明

- robots `secret_hash` 仅存 HMAC 摘要，明文只在创建/轮换接口一次性返回
- rosbridge 客户端为纯 WebSocket（`client.py`），进程内无 ROS/rclpy 依赖
- 协议基线回归：`poc/run_poc.py`（T1-T5）
