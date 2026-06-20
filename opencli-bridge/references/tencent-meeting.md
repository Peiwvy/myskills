# 腾讯会议 (meeting.tencent.com) 逐字稿抓取

## 站点特征

- **opencli 无内置 adapter**（截至 v1.8.3，153 个 site 列表里没有 tencent-meeting）
- **完全走 browser bridge**：复用浏览器已登录的腾讯会议账号 cookie
- **逐字稿页面是 React 虚拟列表**：DOM 里只有可视区 ~17 行，必须分段滚动收集
- 适用页面：`https://meeting.tencent.com/cw/<token>` —— 会议录制 / 纪要详情页

## 前置条件

1. opencli + Browser Bridge 扩展正常（`opencli doctor` 全 OK）
2. 在 Chrome 已登录 `meeting.tencent.com`（会议参与者或被授权人）
3. 知道目标会议的 `cw/<token>` 链接

## 完整流程

### Step 1 — 打开页面

```bash
PROFILE=$(opencli doctor 2>&1 | grep -oP '(?<=  • )\w+(?=: connected)' | head -1)
opencli browser "$PROFILE" open "https://meeting.tencent.com/cw/<TOKEN>"
sleep 3
```

### Step 2 — 切到"逐字稿"标签

页面默认显示"智能总结"。需点击"逐字稿" tab：

```bash
opencli browser "$PROFILE" eval "
(()=>{const b=[...document.querySelectorAll('button')].find(e=>e.textContent.trim()==='逐字稿');b.click();return 'ok';})()
"
```

可选 tab：`纪要` / `时间轴` / `逐字稿`。

### Step 3 — 关键 DOM 结构

逐字稿容器和行结构（2026-06 实测，CSS Modules hash 可能随版本变）：

| 用途 | 选择器 |
|------|--------|
| 滚动容器（虚拟列表） | `.minutes-module-list` |
| 单行 | `.minutes-module-row` |
| 发言人名 | `.paragraph-module_speaker-name__afSbd` |
| 时间码 (mm:ss) | `.minutes-module-p-start-time` |
| 文本 | `.minutes-module-sentences` |

如果 hash 类名变了，回退方案：抓 `.minutes-module-row` 的 `innerText`，按 `\n` 拆出 `[发言人, 时间码, 文本]` 三段。

### Step 4 — 分段滚动收集（避免 eval 超时）

opencli `eval` 单次有超时限制，**不能一把滚到底**。把状态 (`window.__t` Map + `window.__sy` 偏移量) 持久化在页面，shell 循环驱动：

`collect3.js`：

```javascript
(async()=>{
  const c=document.querySelector('.minutes-module-list');
  if(!window.__t) window.__t=new Map();
  const collect=()=>{
    const rows=c.querySelectorAll('.minutes-module-row');
    for(const r of rows){
      const spk=r.querySelector('.paragraph-module_speaker-name__afSbd')?.innerText.trim();
      const ts=r.querySelector('.minutes-module-p-start-time')?.innerText.trim();
      const sent=r.querySelector('.minutes-module-sentences')?.innerText.trim().replace(/\s+/g,' ');
      if(spk && ts && sent){
        const key=ts+'|'+spk;
        if(!window.__t.has(key)) window.__t.set(key,{spk,ts,text:sent});
      }
    }
  };
  const start=window.__sy||0;
  const H=c.scrollHeight;
  const end=Math.min(start+3000,H);
  for(let y=start;y<=end;y+=250){
    c.scrollTop=y;
    await new Promise(r=>setTimeout(r,150));
    collect();
  }
  window.__sy=end;
  return JSON.stringify({collected:window.__t.size,scrolled:end,total:H,done:end>=H});
})()
```

shell 驱动：

```bash
opencli browser "$PROFILE" eval "window.__t=undefined;window.__sy=0;'reset'"

for i in $(seq 1 12); do
  R=$(opencli browser "$PROFILE" eval "$(cat collect3.js)" 2>&1 | tail -1)
  echo "round $i: $R"
  case "$R" in *'"done":true'*) break;; esac
done
```

### Step 5 — 补开头（重要踩坑）

第一轮即便 `scrollTop=0`，虚拟列表的可视区起点不一定是首条。**收集完底部后，强制回顶并补一轮**：

```bash
opencli browser "$PROFILE" eval "
(async()=>{const c=document.querySelector('.minutes-module-list');c.scrollTop=0;await new Promise(r=>setTimeout(r,800));c.scrollTop=0;await new Promise(r=>setTimeout(r,800));return c.querySelectorAll('.minutes-module-row')[0]?.querySelector('.minutes-module-p-start-time')?.innerText;})()
"
# 看到 [00:01] 才算到顶
opencli browser "$PROFILE" eval "window.__sy=0;'reset'"
# 再跑 collect3.js 循环
```

### Step 6 — 导出排序

`dump.js`：

```javascript
(()=>{
  const arr=[...window.__t.values()];
  const toSec=ts=>{const p=ts.split(':').map(Number);return p.length===3?p[0]*3600+p[1]*60+p[2]:p[0]*60+p[1];};
  arr.sort((a,b)=>toSec(a.ts)-toSec(b.ts));
  return arr.map(r=>`[${r.ts}] ${r.spk}: ${r.text}`).join('\n');
})()
```

```bash
opencli browser "$PROFILE" eval "$(cat dump.js)" > transcript.txt
wc -l transcript.txt
```

### Step 7 — 顺便抓"智能总结"

切回 `纪要` tab 后，整页 `document.body.innerText` 已包含 AI 总结（约 2k 字）+ 待办事项，可作为正文上半部分：

```bash
opencli browser "$PROFILE" eval "(()=>{const b=[...document.querySelectorAll('button')].find(e=>e.textContent.trim()==='纪要');b.click();return 'ok';})()"
sleep 1
opencli browser "$PROFILE" eval "document.body.innerText" > summary.txt
```

## 输出文件规范

保存为 markdown，建议位置：`<项目>/1-素材笔记/YYYY-MM-DD-<会议主题>-逐字稿.md`。

frontmatter 至少包含：

```yaml
---
date: YYYY-MM-DD
type: 会议逐字稿
source: 腾讯会议
source_url: https://meeting.tencent.com/cw/<TOKEN>
meeting_id: <会议号>
participants: [...]
duration: 约 X 分钟
state: source/sorted
---
```

正文结构：

1. `## 智能总结（腾讯会议 AI 生成）` — 从 Step 7 提取的结构化重排
2. `## 待办事项` — 转成 `- [ ]` checkbox
3. `## 完整逐字稿` — Step 6 的 `[mm:ss] 发言人: 文本` 逐行

## 踩坑记录

1. **虚拟列表 scrollHeight 会动态增长** — 一开始 H=12038，滚到 9000 时 H 已变成 20776。循环必须每轮重新读 `scrollHeight`，不能预算总轮数。
2. **eval 单次超时** — 一次性 `for(y=0;y<H;y+=300)` 加 await 会触发 "operation aborted"。必须切片（每轮 3000px）。
3. **shell 转义陷阱** — 内联 JS 包含 `;` `<` `>` 时 bash 经常报 `SyntaxError`。**永远把 JS 写到文件，用 `"$(cat file.js)"` 传入**。
4. **变量重复声明** — opencli eval 在同一页面 context 里多次声明 `const c=...` 会报 `Identifier 'c' has already been declared`。每段都包成 IIFE `(()=>{...})()`。
5. **回顶必须 sleep** — `c.scrollTop=0` 后立刻读首行会拿到旧的虚拟窗口（实测仍是 04:42 起）。等 ~800ms 再读才会看到 00:01。
6. **去重 key 必须含时间码** — 同一发言人多条用 `ts+'|'+spk`，光用 spk 会丢条。
7. **cross-origin iframe** — 页面右侧有腾讯元宝 iframe，不需要进去；逐字稿在主 frame。
8. **没有可用的 OpenAPI** — 腾讯会议 cw 页面的 transcript 走的是登录态私有 API，逆向收益低，**browser bridge 抓 DOM 是最稳的路**。

## 已验证

- 2026-06-09 实测：40 分钟会议 / 186 条发言，全量抓取耗时约 30 秒（7 轮滚动 + 1 轮补头）
