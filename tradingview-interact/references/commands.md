# tv CLI 命令参考

工作目录：`~/tools/tradingview-mcp`，所有命令通过 `node src/cli/index.js` 执行。

## 连接与状态

| 命令 | 说明 |
|------|------|
| `tv status` | CDP 连接状态 + 当前图表品种/周期/指标 |
| `tv health_check` | 同 status |
| `tv ensure` | 确保 TV 在运行，如未运行则自动拉起（Desktop 版本用） |
| `tv launch` | 启动 TV Desktop 并开启 CDP（Desktop 版本用） |

## 品种与周期

| 命令 | 说明 |
|------|------|
| `tv symbol <code>` | 切换品种 |
| `tv timeframe <tf>` | 切换周期（1, 5, 15, 60, D, W, M） |
| `tv chart_type <type>` | 切换图表类型 |

## 行情数据

| 命令 | 说明 |
|------|------|
| `tv quote` | 实时报价（last, OHLC, volume） |
| `tv ohlcv --summary` | 近 100 根 K 统计 + 最后 5 根明细 |
| `tv ohlcv --count 50` | 指定 K 线数量 |
| `tv study_values` | 当前所有指标数值 |
| `tv study_values --filter "RSI"` | 指定指标数值 |

## 画线

| 命令 | 说明 |
|------|------|
| `tv draw shape -t horizontal_line -p <价格> --time <戳>` | 水平线 |
| `tv draw shape -t trend_line -p <价1> --time <戳1> -p2 <价2> --time2 <戳2>` | 趋势线 |
| `tv draw shape -t rectangle -p <价1> --time <戳1> -p2 <价2> --time2 <戳2>` | 矩形 |
| `tv draw shape -t text --text "内容" -p <价> --time <戳>` | 文字 |
| `tv draw list` | 列出所有画线 |
| `tv draw get <id>` | 查看画线详情 |
| `tv draw remove <id>` | 删除指定画线 |
| `tv draw clear` | 清除所有画线 |

color overrides: `'{"linecolor":"#FF0000","textcolor":"#FFFFFF"}'`

## 指标

| 命令 | 说明 |
|------|------|
| `tv indicator add "<名称>"` | 添加指标（全名匹配） |
| `tv indicator remove "<名称>"` | 移除指标 |
| `tv indicator list` | 列出当前指标 |

## 截图

| 命令 | 说明 |
|------|------|
| `tv screenshot` | 全屏截图 |
| `tv screenshot -r chart` | 仅图表区域 |
| `tv screenshot -r strategy` | 策略测试区 |
| `tv screenshot -w` | 等待图表渲染稳定后截图 |

## 布局

| 命令 | 说明 |
|------|------|
| `tv pane layout 2x2` | 2×2 网格 |
| `tv pane layout 3x1` | 3×1 横向 |
| `tv pane symbol <index> <code>` | 为第 N 个窗格设品种 |

## Pine Script

| 命令 | 说明 |
|------|------|
| `tv pine set "<代码>"` | 注入代码到编辑器 |
| `tv pine compile` | 编译 |
| `tv pine errors` | 读取编译错误 |
| `tv pine save` | 保存到云 |
