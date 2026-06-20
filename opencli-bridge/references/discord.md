# Discord opencli 使用指南

## 站点特征

- **无 `login` 命令**，走路径二 —— 手动提取 token
- Discord API 被墙，所有操作**必须走代理** `127.0.0.1:7892`
- `opencli discord export` 输出的 `raw_json` 始终为 `null`，大量消息 `content` 为空（实际内容在 embed 结构中）
- 获取完整消息内容需**直接调 Discord API**

## 环境依赖

- **opencli**: 已安装于 `/opt/homebrew/bin/opencli`，通过 `opencli discord` 子命令操作
- **系统代理**: `127.0.0.1:7892`（Clash/V2Ray）
- **Token**: 从本地浏览器提取，保存在 `~/.env` 的 `DISCORD_TOKEN` 字段

---

## 1. 获取 Discord Token

```bash
opencli discord auth --save
```

自动从 Edge/Chrome 浏览器中提取 Discord 用户 token，保存到 `~/.env`。Token 格式为 `MTAwNzg4...`，属于 **user token**（非 bot token），可以调用 Discord 用户 API。

验证登录状态：

```bash
opencli discord status
```

---

## 2. 浏览服务器和频道

### 2.1 列出服务器

```bash
opencli discord dc guilds
```

找到目标服务器名称，如 `美股信息聚合网`。

### 2.2 列出频道

```bash
opencli discord dc channels "美股信息聚合网"
```

输出每个频道的 `id` 和 `name`。

---

## 3. 同步和导出消息

### 3.1 增量同步（只拉新消息）

```bash
opencli discord dc sync <channel_id> --limit 100
```

### 3.2 全量历史消息

```bash
opencli discord dc history --limit 100 <channel_id> \
  --guild-name "美股信息聚合网" \
  --channel-name "频道名"
```

### 3.3 从本地数据库导出

```bash
# 导出为纯文本（⚠️ 只含 content 字段，embed/附件内容缺失）
opencli discord export --hours 96 --format text <channel_id>

# 导出为 JSON
opencli discord export --hours 96 --format json <channel_id>
```

**重要限制**: `opencli discord export` 输出的 `raw_json` 始终为 `null`，且大量消息的 `content` 为空。这是因为消息内容实际存储在 Discord 的 **embed** 结构中，opencli 没有保存。

---

## 4. 直接调用 Discord API（获取完整数据）

### 4.1 为什么需要直接调 API

Discord 消息的真实结构：

```json
{
  "content": "",          // ← opencli 只能拿到这个，经常为空
  "embeds": [{
    "description": "实际内容在这里...",  // ← 真正的文本
    "title": "...",
    "fields": [...]
  }],
  "attachments": [{
    "url": "https://cdn.discordapp.com/...",
    "filename": "image.png"
  }]
}
```

### 4.2 通过代理调用 API

```bash
TOKEN=$(grep DISCORD_TOKEN ~/.env | cut -d= -f2)
PROXY="http://127.0.0.1:7892"

# 拉取频道消息（每次最多100条，支持翻页）
curl -s -x "$PROXY" \
  -H "Authorization: $TOKEN" \
  "https://discord.com/api/v10/channels/<channel_id>/messages?limit=100"

# 翻页：用上次最老消息的 id 作为 before 参数
curl -s -x "$PROXY" \
  -H "Authorization: $TOKEN" \
  "https://discord.com/api/v10/channels/<channel_id>/messages?limit=100&before=<last_msg_id>"
```

### 4.3 Python 示例

```python
import requests

PROXY = "http://127.0.0.1:7892"
TOKEN = "MTAwNzg4..."  # 从 ~/.env 读取
API = "https://discord.com/api/v10"

def fetch_messages(channel_id, limit=100):
    messages = []
    last_id = None
    while len(messages) < 500:
        url = f"{API}/channels/{channel_id}/messages?limit={min(limit, 500 - len(messages))}"
        if last_id:
            url += f"&before={last_id}"
        resp = requests.get(url, headers={"Authorization": TOKEN},
                           proxies={"https": PROXY, "http": PROXY})
        if resp.status_code == 429:  # rate limit
            time.sleep(resp.json()["retry_after"])
            continue
        if resp.status_code != 200:
            break
        data = resp.json()
        if not data:
            break
        messages.extend(data)
        last_id = data[-1]["id"]
        time.sleep(0.3)  # 避免触发限流
    return messages
```

---

## 5. 提取消息中的实际内容

```python
def extract_text(msg):
    texts = []
    # 纯文本
    if msg.get("content", "").strip():
        texts.append(msg["content"])
    # Embed 内容（这才是大头）
    for embed in msg.get("embeds", []):
        desc = embed.get("description", "").strip()
        if desc:
            texts.append(desc)
        for field in embed.get("fields", []):
            texts.append(f"{field.get('name','')}: {field.get('value','')}")
    return "\n".join(texts)
```

---

## 6. 下载附件/图片

```python
def download(url, save_path):
    resp = requests.get(url, proxies={"https": PROXY, "http": PROXY})
    if resp.status_code == 200:
        Path(save_path).write_bytes(resp.content)
```

附件 URL 在 `msg["attachments"][i]["url"]`，同样需要通过代理下载。

---

## 7. 消息过滤要点

从大量消息中提取投资相关信号的关键过滤模式：

### 交易指令（分析师给出）
- 中文模式: `价格 + 动作 + 标的`，如 `423.5出一半avgo`
- 动作词: 出/买/入/接/加仓/止盈/止损/sell put/buy call
- 英文模式: `BTO TICKER STRIKE C/P @PRICE`

### 扫描器信号
- CD底部信号筛选、NX+CD 趋势策略选股
- 格式: `1. CHWY (当前价: 20.82) [30m]`

### 期权告警
- RedAlert/YellowAlert 结构化警告
- 格式: `:RedAlert: NVDA - $230 CALLS 6/12 $1.50 \n STOP LOSS AT $1.20`

### 需要排除的噪音
- 纯数据频道（期权订单流、内部交易、暗池等）——是数据不是建议
- P/L 更新（"up 30%", "50% runners left"）
- 新闻/闲聊/翻译重复

---

## 8. 代理不可用时的备选

若 `127.0.0.1:7892` 代理挂了，检查：

```bash
# 查看系统代理状态
scutil --proxy | grep -E "HTTPEnable|HTTPProxy|HTTPPort"

# 检查 opencli daemon（它使用浏览器扩展连接 Discord）
opencli daemon status

# 重置代理
networksetup -setwebproxy Wi-Fi 127.0.0.1 7892
networksetup -setsecurewebproxy Wi-Fi 127.0.0.1 7892
```

---

## 可用命令速查

| 命令 | 说明 |
|------|------|
| `opencli discord auth --save` | 从浏览器提取并保存 token |
| `opencli discord status` | 验证登录状态 |
| `opencli discord dc guilds` | 列出所有服务器 |
| `opencli discord dc channels <guild>` | 列出服务器下频道 |
| `opencli discord dc sync <channel_id>` | 增量同步消息到本地数据库 |
| `opencli discord dc history <channel_id>` | 全量拉取历史消息 |
| `opencli discord export --hours <n> <channel_id>` | 从本地数据库导出消息 |
| `opencli daemon status` | 检查 daemon 状态 |

## 踩坑记录

1. **`export` 输出残缺** — `raw_json` 始终为 `null`，`content` 常为空，必须直接调 Discord API 获取 embed 内容
2. **必须走代理** — Discord API 被墙，所有 `curl`/`requests` 调用必须带 `-x http://127.0.0.1:7892`
3. **Rate Limit** — API 返回 429 时需 `sleep(retry_after)`，建议每次请求间隔 0.3s
4. **Token 类型** — 是 user token 不是 bot token，权限范围受限于用户账号
