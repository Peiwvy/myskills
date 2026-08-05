---
name: tradingview-interact
description: 通过 Chrome CDP + tradingview-mcp CLI 与 TradingView 交互。支持 Web 版（Chrome CDP 连接网页）和 Desktop 版（MCP 连接本地应用）。用于切换品种、查询行情、获取 OHLCV 数据、画线、管理指标、编写 Pine Script、回测、回放等。当用户要求操作 TradingView 图表时触发。
---

# TradingView 交互

两种接入方式：

| 方式 | 适用版本 | 工具 | 配置 |
|------|----------|------|------|
| **Desktop MCP** | TradingView Desktop 应用 | MCP 工具 (`mcp_call server=tradingview`) | 本地 `tradingview-mcp` 仓库 + `.mcp.json` |
| **Web CLI** | TradingView 网页版 (Chrome) | `tv` CLI (`node src/cli/index.js`) | Chrome CDP 模式 + 打开 tradingview.com |

## Desktop MCP 方式（推荐，已配置）

通过 MCP 直接操控本地 TradingView Desktop 应用。**78 个工具**，覆盖图表、行情、Pine Script、回测、画线、警报、回放、多面板等所有功能。

### 前置条件

- TradingView Desktop 以 CDP 模式运行：`TradingView.exe --remote-debugging-port=9222`
- `tradingview-mcp` 已克隆到 `~/tradingview-mcp` 且 `npm install` 完成
- `.mcp.json` 已配置 `tradingview` server

### 安装与配置

```bash
# 1. 克隆仓库
cd ~ && git clone https://github.com/tradesdontlie/tradingview-mcp.git
cd tradingview-mcp && npm install

# 2. 关闭已有 TradingView，然后以 CDP 模式启动
powershell -Command '$procs = [System.Diagnostics.Process]::GetProcessesByName("TradingView"); foreach ($p in $procs) { $p.Kill() }'
powershell -Command 'Start-Process "<TradingView.exe路径>" -ArgumentList "--remote-debugging-port=9222"'

# 3. 验证端口
Test-NetConnection -ComputerName localhost -Port 9222
```

`.mcp.json` 配置：

```json
{
  "mcpServers": {
    "tradingview": {
      "command": "C:\\Program Files\\nodejs\\node.exe",
      "args": ["C:\\Users\\ztyu\\tradingview-mcp\\src\\server.js"],
      "cwd": "D:\\Documents\\obsidian\\ztyu"
    }
  }
}
```

**注意事项：**
- 每次重启 TradingView Desktop 都需要带 `--remote-debugging-port=9222` 参数
- 内部使用未公开的 TradingView Electron API，版本更新可能 break
- 所有数据处理均在本地，不经过外部服务器
- 不能执行真实交易，仅图表交互

### 常用 MCP 调用

调用格式：`mcp_call(server="tradingview", tool="<工具名>", args={...})`

**连接与状态：**
- `tv_health_check` — 检查 CDP 连接和当前图表状态
- `chart_get_state` — 获取当前图表品种/周期/类型/指标

**品种与行情：**
- `chart_set_symbol({symbol: "AAPL"})` — 切换品种
- `chart_set_timeframe({timeframe: "D"})` — 切换周期
- `quote_get({symbol: "ES1!"})` — 实时报价
- `data_get_ohlcv({count: 100, summary: true})` — K 线数据

**指标：**
- `chart_manage_indicator({action: "add", indicator: "Relative Strength Index", inputs: '{"length": 14}'})` — 添加指标
- `indicator_set_inputs({entity_id: "...", inputs: '{"length": 50}'})` — 调整指标参数
- `data_get_study_values()` — 读取所有指标当前值

**Pine Script：**
- `pine_set_source({source: "<代码>"})` — 注入代码
- `pine_compile()` — 编译
- `pine_smart_compile()` — 智能编译（检测按钮→编译→检查错误→报告变化）
- `pine_get_errors()` — 读取编译错误
- `pine_check({source: "<代码>"})` — 通过服务器端编译验证（无需开图表）
- `pine_analyze({source: "<代码>"})` — 静态分析（数组越界、空引用等）

**画线：**
- `draw_shape({shape: "horizontal_line", point: {time: <戳>, price: <价>}})` — 水平线
- `draw_shape({shape: "trend_line", point: {...}, point2: {...}})` — 趋势线
- `draw_list()` — 列出所有绘图
- `draw_clear()` — 清除所有

**回测：**
- `data_get_strategy_results()` — 策略回测指标
- `data_get_trades({max_trades: 50})` — 交易记录
- `data_get_equity()` — 权益曲线

**回放：**
- `replay_start({date: "2024-01-15"})` — 开始回放
- `replay_step()` — 前进一根 K 线
- `replay_stop()` — 停止回放

**多面板：**
- `pane_set_layout({layout: "2x2"})` — 设置布局
- `pane_set_symbol({index: 0, symbol: "ES1!"})` — 为窗格设品种

**自选：**
- `watchlist_get()` — 读取自选列表
- `watchlist_add({symbol: "AAPL"})` — 添加到自选

**截图：**
- `capture_screenshot({region: "chart"})` — 截取图表区域

**UI 操控：**
- `ui_open_panel({panel: "pine-editor", action: "toggle"})` — 开关面板
- `ui_keyboard({key: "Escape"})` — 按键

完整工具列表见 [[TradingView#Desktop MCP 配置（2026-06-24 已配好）|TradingView 笔记]]。

## Web CLI 方式（备选）

通过 Chrome CDP 连接 TradingView 网页版，用 `tv` CLI 控制。

### 前置条件

Chrome 必须以 CDP 模式运行，且已打开 tradingview.com 页面：

```bash
~/bin/tv-chrome-cdp.sh
```

验证连接：
```bash
cd ~/tools/tradingview-mcp && node src/cli/index.js status
```

所有 CLI 命令的工作目录：`~/tools/tradingview-mcp`，入口 `node src/cli/index.js`（下文简称 `tv`）。

### 常用操作

#### 切换品种

```bash
tv symbol <code>
```

- A 股直接传代码：`688017`、`600519`
- 美股传代码：`AAPL`、`TSLA`
- 港股传代码：`0700`、`9988`
- 期货/外汇：`ES1!`、`BTCUSD`

#### 查询行情

```bash
tv quote            # 当前图表品种的实时报价 (OHLCV + last)
tv status           # 图表状态：品种、周期、指标列表
tv ohlcv --summary  # 最近 100 根 K 的统计 + 最后 5 根明细
```

#### 画线

```bash
# 水平线（需 --time 参数，取最近一根 K 的时间戳即可）
tv draw shape -t horizontal_line -p <价格> --time <unix时间戳>

# 红色水平线
tv draw shape -t horizontal_line -p <价格> --time <时间戳> --overrides '{"linecolor":"#FF0000"}'

# 趋势线（两点连线）
tv draw shape -t trend_line -p <价格1> --time <时间戳1> -p2 <价格2> --time2 <时间戳2>

# 矩形
tv draw shape -t rectangle -p <价格1> --time <时间戳1> -p2 <价格2> --time2 <时间戳2>

# 文字标注
tv draw shape -t text --text "标注内容" -p <价格> --time <时间戳>

# 查看/删除
tv draw list
tv draw clear     # 删除所有
```

**画线注意事项：**
- `--time` 参数必填，从 `tv ohlcv --summary` 的 `last_5_bars` 中取最后一条的 time
- 画线前先用 `tv status` 确认当前图表是目标品种
- 线画上去后让用户在 Chrome 里确认是否可见

#### 指标操作

```bash
tv indicator add "RSI"           # 添加指标
tv indicator remove "RSI"        # 移除指标
tv indicator list                # 列出当前指标
```

#### 截图

```bash
tv screenshot                    # 全屏截图
tv screenshot -r chart           # 仅图表区域
```

### 排查

- **CDP 连不上**：Chrome 是否以 `--remote-debugging-port=9222` 启动？运行 `~/bin/tv-chrome-cdp.sh`
- **API 不可用**：TradingView 页面是否完全加载？等几秒重试
- **画线看不到**：确认 `--time` 参数正确，确认当前图表品种正确（`tv status`）
- **切品种失败**：TradingView 搜索框可能不支持该代码格式，尝试在 Chrome 里手动输入验证

完整命令参考见 [commands.md](references/commands.md)。
