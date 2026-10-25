# blender_bridge · S0 直挂联调手册

> S0 目标：验证「提示词 → MCP 分步建模 → 视口截图 → 人工确认 → 导出」最小闭环。
> 本阶段不做白名单审批门、不建库、不做管理页（S1 落地）。

## 1. 前置环境（一台 Windows 工作站即可）

1. **Blender 4.x**（建议 4.2+）已安装并可正常启动 GUI。
2. **uv / uvx** 可用：`uv --version`。没有则先装 uv；`mcp-for-blender` 由 `uvx --python 3.11` 临时拉取，无需全局 pip 安装。
3. 安装官方 addon（mcp-for-blender 配套的 Blender 端脚本）：
   - 取官方 addon 脚本（ahujasid/blender-mcp 仓库的 `addon.py`）；
   - Blender → Edit → Preferences → Add-ons → Install 选中该脚本 → 勾选启用；
   - 每次建模前，在 3D 视图 N 面板找到 MCP 面板，点 **Start MCP Server**（默认监听 127.0.0.1:9876）。

## 2. 启用插件

1. 把 `plugins/blender_bridge/` 整体放入仓库 `plugins/`（本仓库已就位）。
2. 在 VeroRun 插件管理页安装/启用 `Blender Bridge`。
3. 内核会在 install 时自动把 `plugin.json` 的 `mcp_servers` 同步进 `plugin_mcp_servers`，
   工具对 Agent 暴露为 `mcp__blender_bridge__blender__<tool>`，插件代码无需注册。

## 3. 连通验证（按序排查）

1. Blender GUI 已开、addon 已 Start MCP Server（端口 9876）。
2. 在会话里让 Agent 调 `get_scene_info`：
   - 返回当前场景对象列表 → 链路通；
   - 超时/连接拒绝 → 见第 5 节。
3. 再调 `get_viewport_screenshot` 能取回图 → 视觉通道通。
4. 跑一个最小动作：`create_object` 建一个立方体 → `get_object_info` 读回 → `export_scene` 前先停下，等人工确认。

## 4. S0 测试集（10 条，用于采集基线）

1. 产品展示台：一个简单台座 + 主体几何体，三点布光。
2. 低多边形山地场景（几座锥体/斜面 + 地面）。
3. Logo 立体化为一个挤出文字/标志体。
4. 节日氛围小场景（地面 + 几个装饰几何体，不接外部素材）。
5. 材质球展示台（同几何换 2–3 种材质对比）。
6. 极简立方体组合（错落堆叠，讲比例与间距）。
7. 几何抽象构图（球体/圆环/柱体构成的静帧）。
8. 参考图复刻（给一张静物图，Agent 只搭几何骨架，不追求像素）。
9. HDRI 环境下的单主体（若已配 Poly Haven 则用，未配跳过）。
10. 静态站位展示（一个占位体 + 相机环绕角度设定，不做动画）。

每条记录：首评是否通过、修订轮次、端到端耗时、失败归类（连接/超时/建模/评审/导出）。

## 5. 常见失败

| 现象 | 原因 | 处理 |
|---|---|---|
| `spawn uvx ENOENT` | 子进程 PATH 找不到 uvx | plugin.json `command` 改为 uvx 绝对路径，或改 `cmd /c uvx` |
| 连接拒绝 / 超时 | Blender addon 没 Start，或端口不一致 | N 面板重启 MCP Server；核对 `blender_port` |
| 首条命令偶发失败 | 官方已知 quirk | 重试一次即可 |
| 复杂步骤超时 | 单步体量过大 | 拆成更小步；`mcp_timeout_sec` 最大调到 300 |
| 无头 Windows 无法 EEVEE 渲染 | EEVEE 纯 GPU，无头不支持 | S0 用 GUI；生产走 Linux Cycles/EGL（S2） |
| 多个人同时连 | 官方单 socket 单客户端 | S0 串行使用；S2 上多实例多端口 |

## 6. 边界（S0 不做）

- 不做精确 CAD（尺寸/公差/STEP）——属 cad_bridge。
- `execute_blender_code` 任意代码、`export_scene` 交付、外部付费 API，S0 一律先人工确认再执行（审批门在 S1 自动化）。
- 不建任务/工件表，不落管理页；凭证（Sketchfab/Hyper3D/混元）按需在系统配置填写，代码内不出现任何 key。

## 7. 遥测与许可

- `DISABLE_TELEMETRY=true` 已在 mcp_servers.env 声明，关闭官方匿名遥测。
- mcp-for-blender 为 MIT；Blender GPL 以独立进程调用，不把其代码拷入分发本体。
