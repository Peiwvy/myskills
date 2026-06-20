# 小宇宙 (xiaoyuzhou) opencli 配置

## 站点特征

- 没有 `login` 命令
- 需要手动创建 `~/.opencli/xiaoyuzhou.json`
- 适配器通过 HTTP API 调用，不走浏览器
- token 不是标准 OAuth2，而是 JWT（HS256 加密）

## 凭证文件格式

```json
{
  "access_token": "<x-jike-access-token>",
  "refresh_token": "<x-jike-refresh-token>"
}
```

保存在 `~/.opencli/xiaoyuzhou.json`。

## 获取 token 的方法

### 方法一：从浏览器 Cookie 提取（推荐）

小宇宙的 token 存在 cookie 里，key 是 `x-jike-access-token` 和 `x-jike-refresh-token`。但 `www.xiaoyuzhoufm.com` 网页版没有登录入口，需要登录以下域名之一：

- `h5.xiaoyuzhoufm.com` — H5 版（需在 App 的 WebView 里登录）
- **`podcaster.xiaoyuzhoufm.com` — 播客创作者后台（推荐）** ← 网页版有登录入口

#### 步骤

**1. 在浏览器登录 `podcaster.xiaoyuzhoufm.com`**

打开 Chrome/Edge，访问 `https://podcaster.xiaoyuzhoufm.com/`，按提示登录（微信扫码 / 手机号）。

**2. 用 opencli browser 提取 token**

```bash
# 创建浏览器 session 并打开已登录页面
opencli browser xyz open https://podcaster.xiaoyuzhoufm.com/podcast

# 抓取 cookie
opencli browser xyz eval "document.cookie"
```

**3. 解析 cookie 并写入配置**

cookie 示例：
```
x-jike-access-token=eyJhbGciOiJIUzI1NiIs...; x-jike-refresh-token=eyJhbGciOiJIUzI1NiIs...
```

用 Python 提取：

```python
import json, re

cookie = '<上一步的 cookie 字符串>'
access_token = re.search(r'x-jike-access-token=([^;]+)', cookie).group(1)
refresh_token = re.search(r'x-jike-refresh-token=([^;]+)', cookie).group(1)

with open(os.path.expanduser('~/.opencli/xiaoyuzhou.json'), 'w') as f:
    json.dump({"access_token": access_token, "refresh_token": refresh_token}, f, indent=2)
```

### 方法二：从 macOS App 的 WKWebView 提取（备选）

如果网页版无法登录，可以从 macOS 小宇宙 App 的 WebView 存储中提取：

```bash
# 找到 LocalStorage 中的 _l_KPLiPs（这是 App 内部 session，不是 opencli 需要的 token）
# 这个值不是 access_token，opencli 无法直接使用
# 建议使用方法一
```

> **注意**：App 里的 `_l_KPLiPs` 不是 opencli 需要的 access_token，opencli 需要的是 JWT 格式的 token（`eyJ...` 开头，三段 base64 用点号连接）。

## 可用命令

| 命令 | 说明 |
|------|------|
| `opencli xiaoyuzhou podcast <id>` | 查看播客信息 |
| `opencli xiaoyuzhou podcast-episodes <id>` | 列出播客单集 |
| `opencli xiaoyuzhou episode <id>` | 查看单集详情 |
| `opencli xiaoyuzhou download <id>` | 下载单集音频 |
| `opencli xiaoyuzhou transcript <id>` | 下载单集逐字稿（JSON + 文本） |

Episode ID 从 URL 中提取：`https://www.xiaoyuzhoufm.com/episode/<id>`

## Token 过期处理

token 有过期时间（JWT 中 `iat` 字段）。过期后 opencli 会尝试用 `refresh_token` 刷新，返回 401 时需要重新获取：

```bash
# 症状：opencli 返回 401
# 解决：重新执行"获取 token"步骤，更新 ~/.opencli/xiaoyuzhou.json
```

## 批量下载字幕

当用户要求从播客页面批量下载字幕时，按以下规范执行。

### 保存位置

- **先询问用户**想保存在哪个目录，如果用户没有指定则默认保存到**当前工作目录**
- 如果目录不存在，自动创建

### 文件格式要求

- **只保留 `.txt`**，不要 JSON
- **扁平存放**，不要子目录（每个 episode 一个子目录是 opencli 默认行为，需要 flatten）
- **命名规则**：`{期数}-{副标题}.txt`

### 标题转换规则

小宇宙标题格式：`《播客名》（期数）副标题`

去掉 `《播客名》（` 和 `）`，把 `）` 替换为 `-`，得到 `期数-副标题.txt`。

示例：
- `《见证逆潮》（86）中国的道路-双顺差的密码` → `86-中国的道路-双顺差的密码.txt`
- `《见证逆潮》（102）K型之殇-货币洪水往哪里流` → `102-K型之殇-货币洪水往哪里流.txt`

如果无法从标题自动构造文件名，需要找用户确认。

### 操作流程

1. **确定保存位置**：询问用户，未指定则用当前目录
2. **提取播客 ID**：从 URL `https://www.xiaoyuzhoufm.com/podcast/<id>` 提取
3. **列出所有单集**：`opencli xiaoyuzhou podcast-episodes <id> --limit 200 -f json`
4. **缺失的单集从 URL 补**：如果某期不在列表里，让用户提供该集的 URL，从中提取 eid，用 `opencli xiaoyuzhou episode <eid>` 确认标题
5. **逐个下载并重命名**：
   ```bash
   tmpdir=$(mktemp -d)
   opencli xiaoyuzhou transcript <eid> --output "$tmpdir" -f json
   txt=$(find "$tmpdir" -name "transcript.txt" -type f)
   cp "$txt" "<目标目录>/<期数>-<副标题>.txt"
   rm -rf "$tmpdir"
   ```
6. **最终确认**：`ls -1` 目标目录，确认文件名和数量

### 注意事项

- `podcast-episodes` API 可能不返回全部历史单集，此时需要用户提供缺失单集的直接链接

## 踩坑记录

1. **`www.xiaoyuzhoufm.com` 没有登录入口** — 只有"下载 App"引导页，不要在这个域名浪费时间
2. **`_l_KPLiPs` 不是 access_token** — 这是 App 内部 session key，opencli 不认，必须用 cookie 里的 JWT
3. **access_token 和 refresh_token 必须同时存在** — opencli 会校验两个字段都不能为空
4. **token 是 HS256 JWT，payload 里的 `data` 字段是 AES 加密的实际凭证** — 不要试图解析，直接原样使用
