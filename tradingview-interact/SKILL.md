---
name: tradingview-interact
description: 通过 Chrome CDP + tradingview-mcp CLI 与 TradingView Web 端交互。用于在图表上切换品种、查询行情、获取 OHLCV 数据、画水平线/趋势线/矩形、管理指标等。当用户要求操作 TradingView 图表（切换标的、查价格、画线、加指标、看 K 线数据）时触发。
---

# TradingView 交互

通过 Chrome DevTools Protocol 连接 TradingView Web 端，用 `tv` CLI 控制图表。

## 前置条件

Chrome 必须以 CDP 模式运行，且已打开 tradingview.com 页面：

```bash
~/bin/tv-chrome-cdp.sh
```

验证连接：
```bash
cd ~/tools/tradingview-mcp && node src/cli/index.js status
```

所有 CLI 命令的工作目录：`~/tools/tradingview-mcp`，入口 `node src/cli/index.js`（下文简称 `tv`）。

## 常用操作

### 切换品种

```bash
tv symbol <code>
```

- A 股直接传代码：`688017`、`600519`
- 美股传代码：`AAPL`、`TSLA`
- 港股传代码：`0700`、`9988`
- 期货/外汇：`ES1!`、`BTCUSD`

### 查询行情

```bash
tv quote            # 当前图表品种的实时报价 (OHLCV + last)
tv status           # 图表状态：品种、周期、指标列表
tv ohlcv --summary  # 最近 100 根 K 的统计 + 最后 5 根明细
```

### 画线

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

### 指标操作

```bash
tv indicator add "RSI"           # 添加指标
tv indicator remove "RSI"        # 移除指标
tv indicator list                # 列出当前指标
```

### 截图

```bash
tv screenshot                    # 全屏截图
tv screenshot -r chart           # 仅图表区域
```

## 排查

- **CDP 连不上**：Chrome 是否以 `--remote-debugging-port=9222` 启动？运行 `~/bin/tv-chrome-cdp.sh`
- **API 不可用**：TradingView 页面是否完全加载？等几秒重试
- **画线看不到**：确认 `--time` 参数正确，确认当前图表品种正确（`tv status`）
- **切品种失败**：TradingView 搜索框可能不支持该代码格式，尝试在 Chrome 里手动输入验证

完整命令参考见 [commands.md](references/commands.md)。
