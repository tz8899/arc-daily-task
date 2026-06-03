---
name: arc-daily-task
description: "在 Arc Community (community.arc.network) 自动完成每日积分任务 — 阅读 5 篇博客（10 分）+ 观看 4 个视频（16 分），最多 26 分。通过 chrome-devtools MCP 控制浏览器执行。Use when user says 跑 arc 任务 / 做 arc 每日积分 / arc 签到 / arc daily task。"
user-invocable: true
---

# Arc Community 每日任务

通过 chrome-devtools MCP 自动完成 Arc Community 积分任务：阅读 5 篇内容 + 观看 1 个视频 + 触发签到 ≈ 35 分。

## 浏览器环境（建议，非强制）

**优先使用 MCP 默认连接**。只要 `mcp__chrome-devtools__list_pages` 能正常返回页面列表，就直接用，不要动用户当前的 profile/端口。

**默认连接不可用时**才回退 — 优先按用户全局 CLAUDE.md 里的浏览器规则处理（如果定义了独立调试浏览器的端口/启动脚本，按那里走）；没有就直接告诉用户 MCP 连不上，让用户检查 chrome-devtools MCP 配置或启动浏览器。

执行前判定流程：
1. 直接调用 `chrome-devtools` MCP 工具的 list_pages / 相关方法 — 能正常连接就走默认连接，跳到下一节
2. 失败 → 提示用户运行 skill 目录下的 `launch-chrome-debug.bat`（或 `launch-chrome-debug.ps1`）启动独立调试 Chrome
3. 仍连不上：告诉用户 MCP 连接异常，请检查 chrome-devtools MCP 配置或重启 Claude Code 会话

**禁止**：Playwright、pkill 用户日常浏览器、未经允许切换默认 profile。

## 大前提 — 必须先登录

**任何任务步骤之前**，必须先验证当前 MCP 连接的浏览器已登录 Arc Community：
1. 调用 `mcp__chrome-devtools__navigate_page` 打开 `https://community.arc.network/home`
2. 等待 3 秒，调用 `evaluate_script`：
   ```js
   () => ({
     loggedIn: !![...document.querySelectorAll('header img, nav img')].find(i => /\/avatar\//i.test(i.src || ''))
   })
   ```
   （用户头像 src 含 `/avatar/` 路径；alt 是用户名而非 "avatar"，所以不能用 alt 匹配）
3. **未登录** → **立即停止任务**，输出以下提示并结束（不要尝试任何后续步骤）：

   ```
   ❌ 当前浏览器未登录 Arc Community

   登录方式（必须人工操作，无法自动化）：
   1. 切到我刚才打开的浏览器窗口
   2. 在该窗口里手动登录 Arc Community
   3. 登录成功后回来跟我说"再试一次"

   说明：登录态会持久保存到该 profile，下次自动加载，不用每天登录。
        如果用的是独立调试 Chrome（标题栏显示 "Arc House"），它与你日常 Chrome 隔离，
        所以即使日常 Chrome 已登录，这里也需要单独登录一次。
   ```
4. **已登录** → 继续后续步骤

⚠️ 不要尝试自动填写登录表单、不要建议用 cookies/token 注入、不要主动切换 profile —— 用户必须自己操作浏览器登录。

## 状态哲学：无本地文件

本 skill **不持久化任何状态**。"已做"与否以 Arc my-contributions 页面为唯一权威来源——每次启动都实时抓取页面的 `readTitles` / `videoTitles` 作为排除集。

好处：
- 做了但 Arc 没给分的任务会被**自动重试**（不会进本地黑名单永远跳过）
- 无 last_run 文件 → 无需"今天是否已执行"的前置检查
- 并发 / 今日重复触发的保护应由调用方（如 launchd + run.sh 的 lockfile）承担，skill 不做

唯一前置检查：**登录状态**（见上面"大前提"章节）。

---

## 公用脚本：滚动加载

每步多次复用，统一参数：
```js
async (maxRounds = 30, stableTarget = 2) => {
  await new Promise(r => setTimeout(r, 3000));
  let lastH = 0, stable = 0;
  for (let i = 0; i < maxRounds; i++) {
    window.scrollTo(0, document.body.scrollHeight);
    await new Promise(r => setTimeout(r, 1500));
    const btn = [...document.querySelectorAll('button')].find(b => /^load more$/i.test(b.textContent.trim()));
    if (btn) { btn.scrollIntoView(); await new Promise(r => setTimeout(r, 500)); btn.click(); await new Promise(r => setTimeout(r, 2000)); stable = 0; continue; }
    const h = document.body.scrollHeight;
    if (h === lastH) { if (++stable >= stableTarget) return { rounds: i, status: 'stable' }; }
    else { stable = 0; lastH = h; }
  }
  return { rounds: maxRounds, status: 'max' };
}
```

调用时把参数嵌入：`(async () => { /* 上面的函数体，maxRounds=40, stableTarget=2 */ })()`。

---

## 执行步骤

### 第一步 — 抓已浏览历史（按 title）

⚠️ **关键约束**：my-contributions 页面**只有标题文本，没有指向内容的 `<a href>`**（实测确认）。所以这一步**必须用 title 比对**，不能用 slug。这是整个 skill 唯一用 title 的地方，其他步骤一律 slug。

调用顺序：
1. `navigate_page` → `https://community.arc.network/home/contributors/my-contributions`
2. `evaluate_script` 跑滚动加载脚本（`maxRounds=40, stableTarget=2`）
3. `evaluate_script` 跑提取脚本：
```js
() => {
  const main = document.querySelector('main');
  const text = main.innerText;
  const readTitles = [], videoTitles = [];
  const lines = text.split('\n');
  for (let i = 0; i < lines.length; i++) {
    if (lines[i].includes('Read Content') || lines[i].includes('Watch a Video')) {
      for (let j = i + 1; j < Math.min(i + 5, lines.length); j++) {
        if (lines[j].includes(' · ')) {
          const title = lines[j].split(' · ')[1];
          if (title && title.trim()) {
            if (lines[i].includes('Read Content')) readTitles.push(title.trim());
            else videoTitles.push(title.trim());
          }
          break;
        }
      }
    }
  }
  return { readTitles, videoTitles };
}
```

格式假设：`Read Content\n<count>\n<date> · <title>\n+\n<points>`，依赖 ` · ` 分隔符。

这两个数组（`readTitles` / `videoTitles`）直接作为第三步筛选的排除集，不读也不写任何本地文件。

### 第二步 — 加载首页全部内容并提取候选

1. `navigate_page` → `https://community.arc.network/home`
2. `evaluate_script` 跑滚动加载（`maxRounds=30, stableTarget=2`）
3. `evaluate_script` 提取候选（用 href 定位 + 父卡片找标题；链接文本常是 "Replay" 不可靠）：
```js
() => {
  const links = [...document.querySelectorAll('a[href]')];
  const items = [], seen = new Set();
  for (const a of links) {
    const href = a.getAttribute('href') || '';
    const m = href.match(/\/home\/(blogs|resources|externals|videos)\/([^/?#]+)/);
    if (!m) continue;
    const slug = `${m[1]}/${m[2]}`;
    if (seen.has(slug)) continue;
    seen.add(slug);
    let card = a, found = null;
    for (let d = 0; d < 8 && card; d++, card = card.parentElement) {
      const lines = (card.innerText || '').split('\n').map(s => s.trim()).filter(Boolean);
      if (lines.length < 2) continue;
      const type = lines.find(l => ['Video','Blog','Resource','External Content'].includes(l));
      const title = lines.find(l => l.length >= 5 &&
        !['Video','Blog','Resource','External Content','Replay','Read more','Watch'].includes(l) &&
        !/^\d+:\d+$/.test(l));
      if (type && title) { found = { type, title: title.slice(0, 120), slug }; break; }
    }
    if (found) items.push(found);
  }
  return items;
}
```

### 第三步 — 筛选未浏览

按 **title** 比对（第一步抓到的 `readTitles` / `videoTitles` 是排除集）：
- `title` 在 `videoTitles` Set → 跳过
- `title` 在 `readTitles` Set → 跳过
- `type === 'External Content'` → **跳过**（实测不触发 Read Content 计分，详见"关键注意"）
- `type === 'Video'` → 加入 `unwatch_videos`（保留完整对象 `{type, title, slug}`，slug 用于第四/五步访问 URL）
- 其他（Blog / Resource）→ 加入 `unread_articles`

排序：`Blog > Resource`（External 已在上面过滤掉）。

如果 `unread_articles.length < 5` 或 `unwatch_videos.length < 4`，能做几个做几个，最后报告里说明。

### 第四步 — 阅读最多 5 篇文章

对 `unread_articles` 顺序遍历，记录 `success_articles = []`。终止条件：成功 5 篇 / 连续失败 3 篇 / 列表读完。

每篇执行：
1. `navigate_page` → `https://community.arc.network/home/${article.slug}`（15s 超时；超时算失败）
2. `evaluate_script` 滚动模拟阅读（≈20 秒/篇）：
```js
async () => {
  await new Promise(r => setTimeout(r, 2000));
  const h = document.body.scrollHeight;
  for (let i = 1; i <= 8; i++) {
    window.scrollTo({ top: (h * i) / 9, behavior: 'smooth' });
    await new Promise(r => setTimeout(r, 2000));
  }
  await new Promise(r => setTimeout(r, 2000));
  return { title: document.title };
}
```
3. 判断成功：`title` 非空且不含 `404` / `error` / `not found`（不区分大小写）→ 推入 `success_articles`

### 第五步 — 观看 4 个视频

对 `unwatch_videos` 顺序尝试，记录 `success_videos = []`。终止条件：成功 4 个 / 连续失败 3 个 / 列表读完。

每个视频执行：
1. `navigate_page` → `https://community.arc.network/home/${video.slug}`
2. `mcp__chrome-devtools__wait_for` 等待页面稳定（5s 超时）或直接 `evaluate_script` 等 3s
3. `take_snapshot` 找到 `role=button` 且 name 匹配 `/^Play [Vv]ideo/`（**Wistia 是 "Play Video" 大写 V，YouTube 是 "Play video" 小写 v**，两者都要兼容；Wistia 还可能有多个同名按钮，挑第一个即可），拿到 uid
4. 如果没找到 button：`evaluate_script` 试 iframe 兜底：
```js
() => {
  const f = document.querySelector('iframe[src*="wistia"], iframe[src*="player"], iframe[allow*="autoplay"]');
  if (f) { f.click(); return 'clicked-iframe'; }
  return 'not-found';
}
```
5. 如果 take_snapshot 拿到了 uid，调用 `click(uid)` 点击；如果只能点 iframe，用上面的 evaluate_script
6. 等 **40 秒**（`evaluate_script` 内 setTimeout）
7. 可选：截图确认播放（检查没有"Play"按钮叠加层），成功则推入 `success_videos`，继续播放下一个

### 第六步 — 报告

```
执行完成！
- 成功阅读 X/5 篇：1. <title> 2. <title> ...
- 观看视频 Y/4 个：1. <title> 2. <title> ...
- 预计积分：{X*2 + Y*4} 分
- 剩余库存：文章 N 篇 / 视频 N 个
- 5 分钟后到账，到 my-contributions 页面确认
```

---

## 关键注意

| 项 | 说明 |
|---|------|
| 主键 | 用 **title**（my-contributions 页面无 link，只能取 title）；slug 仅用于访问 URL。同一来源的 title 通常一致，标点失配是潜在风险但实测罕见 |
| 时区 | `datetime.now(timezone.utc).date()`，避免跨日 |
| 滚动加载 | my-contributions 默认只显示一页，必须滚到 `stable >= 2` |
| External Content | **第三步直接跳过**：2026-04-17/18 连续三次共读 5 篇 External（20 秒停留），均未在 my-contributions 产生 Read Content 活动、未加 25 分。当天 5 Blog/Resource 读的是正常计分。结论：External 不走 Read Content 计分路径，skill 不再尝试 |
| 视频播放 | 优先 take_snapshot 找 `Play video` 按钮（用动态 uid，不要写死）；iframe.click 兜底；点击后 `evaluate_script` 内 setTimeout ≈ 40 秒 |
| 文章模拟 | 滚动 8 段共 ≈20 秒 |
| 无本地状态 | skill 本身无任何 read/write 本地文件 — 所有"已做"判定实时来自 my-contributions 页面 |

## 错误处理

| 错误 | 处理 |
|------|------|
| MCP 默认连接失败 | 回退探活 9333；仍失败提示用户运行 `launch-chrome-debug.bat`（在 skill 目录下）或手动启动 Chrome 调试实例；都不行提示用户检查 MCP 配置或重启 Claude Code |
| 未登录 | 停止，提示在当前浏览器窗口里登录一次 |
| 文章 navigate 超时 / title 含 error | 跳过下一篇；连续 3 次失败终止阅读阶段 |
| 视频找不到播放按钮 | 换下一个视频，最多试 3 个；都失败则跳过视频任务 |
| 候选库存不足 | 能做几个做几个，结果报告里说明 |
