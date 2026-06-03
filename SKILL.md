---
name: arc-daily-task
description: "在 Arc Community (community.arc.network) 自动完成每日积分任务 — 阅读 5 篇博客（10 分）+ 观看 4 个视频（16 分），最多 26 分。通过 chrome-devtools MCP 控制浏览器执行。Use when user says 跑 arc 任务 / 做 arc 每日积分 / arc 签到 / arc daily task。"
user-invocable: true
---

# Arc Community 每日积分任务

<!--
  ════════════════════════════════════════════════════════════
  技能概述
  ════════════════════════════════════════════════════════════
  本技能通过 chrome-devtools MCP 控制 Chrome 浏览器，自动完成 
  Arc Community 每日积分任务。
  
  当前规则（2026-06 更新）：
    - 阅读 5 篇博客 → 10 分（每篇 +2）
    - 观看 4 个视频 → 16 分（每个 +4）
    - 每日签到 → +1
    - 每日上限 ≈ 26~27 分
  
  注意：规则可能会随时间调整，具体以 Arc 官方公布为准。
  ════════════════════════════════════════════════════════════
-->

## 1. 浏览器连接检测

<!--
  目的：判断 chrome-devtools MCP 是否可用。
  流程：
    1. 优先使用已配置好的 MCP 连接
    2. 如果连不上，提示用户运行启动脚本
    3. 禁止使用 Playwright、pkill 等可能影响用户日常浏览器的操作
-->

1. 调用 `chrome-devtools` MCP 工具的 list_pages / 相关方法，检测是否能正常连接
2. 连接成功 → 跳到下一节（登录检查）
3. 连接失败 → 提示用户运行 `launch-chrome-debug.bat`（位于 skill 目录下）
4. 仍连不上 → 提示检查 MCP 配置或重启 Claude Code

## 2. 登录状态验证

<!--
  目的：确保浏览器已登录 Arc Community，否则后续操作无效。
  原理：Arc 页面的 header/nav 中，已登录用户的头像 img src 包含
        "/avatar/" 路径，通过检测这个特征判断登录态。
  
  注意：
    - 登录必须由用户手动完成（不能自动填充表单）
    - 登录态保存在独立 Chrome profile 中，后续不用重复登录
    - 独立调试 Chrome 与用户日常浏览器隔离，需分别登录
-->

1. 导航到 `https://community.arc.network/home`
2. 等待 3 秒后，执行以下 JS 检测头像元素：
   ```js
   () => ({
     loggedIn: !![...document.querySelectorAll('header img, nav img')]
       .find(i => /\/avatar\//i.test(i.src || ''))
   })
   ```
   <!-- src 含 "/avatar/" 即视为已登录 -->
3. **未登录** → 停止任务，提示用户手动登录后重试
4. **已登录** → 继续执行

<!-- ⛔ 禁止自动填写登录表单、注入 cookies/token 或切换 profile -->

## 3. 获取已浏览历史（作为排除集）

<!--
  目的：读取 my-contributions 页面，获取今天已经做过的内容，
        避免重复阅读/观看。
  
  关键约束：
    - my-contributions 页面只有标题文本（title），没有链接（href）
    - 所以必须用 title 字符串比对，不能用 URL slug
    - 这是整个技能唯一用 title 的地方，后续步骤都用 slug
  
  页面格式假设：
    Read Content
    x1
    May 30th, 2026 · 文章标题
    +
    2
-->

1. 导航到 `https://community.arc.network/home/contributors/my-contributions`
2. 执行滚动加载脚本，确保所有历史记录都加载出来（最多 40 轮）
3. 提取已读标题和已看视频标题：
   ```js
   () => {
     const main = document.querySelector('main');
     const text = main.innerText;
     const readTitles = [], videoTitles = [];
     const lines = text.split('\n');
     for (let i = 0; i < lines.length; i++) {
       // 找到 "Read Content" 或 "Watch a Video" 标记行
       if (lines[i].includes('Read Content') || lines[i].includes('Watch a Video')) {
         // 在标记行后面 5 行内找 "日期 · 标题" 格式
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
4. 这两个数组直接用作第三步的排除集，**不写任何本地文件**

## 4. 加载首页并提取候选内容

<!--
  目的：从 Arc Community 首页获取所有可用的博客和视频列表。
  
  逻辑：
    - 滚动加载全部内容（处理 "Load More" 按钮）
    - 提取每个内容的类型（Blog/Resource/Video）、标题（title）和路径（slug）
    - slug 用于后续导航到具体页面
    - title 用于与已做历史比对去重
  
  注意：
    - External Content 类型的内容不计分，虽然提取但后续会过滤掉
    - 卡片中的链接文本常是 "Replay" 不可靠，所以从父级 DOM 找标题
-->

1. 导航到 `https://community.arc.network/home`
2. 滚动加载全部内容（最多 30 轮，连续 2 次高度不变视为加载完成）
3. 提取所有候选内容：
   ```js
   () => {
     const links = [...document.querySelectorAll('a[href]')];
     const items = [], seen = new Set();
     for (const a of links) {
       const href = a.getAttribute('href') || '';
       // 匹配 /home/blogs/xxx, /home/resources/xxx, /home/externals/xxx, /home/videos/xxx
       const m = href.match(/\/home\/(blogs|resources|externals|videos)\/([^/?#]+)/);
       if (!m) continue;
       const slug = `${m[1]}/${m[2]}`;
       if (seen.has(slug)) continue; // 去重
       seen.add(slug);
       
       // 向上遍历最多 8 层父级，找卡片容器中的类型和标题
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

## 5. 筛选未浏览的内容

<!--
  目的：从提取的候选列表中，排除已做过的内容，得到今天要做的清单。
  
  筛选规则：
    1. title 在 readTitles 中 → 已读过，跳过
    2. title 在 videoTitles 中 → 已看过，跳过
    3. type 为 External Content → 实测不计分，跳过
    4. type 为 Video → 加入待看视频列表（unwatch_videos）
    5. 其他（Blog/Resource）→ 加入待读文章列表（unread_articles）
  
  排序：Blog 优先于 Resource（Blog 更稳定计分）
  
  库存不足时：能做几个做几个，最后报告说明
-->

1. 用 title 逐一比对排除
2. 剩余内容分类为 `unread_articles`（Blog/Resource）和 `unwatch_videos`（Video）
3. 如果不足 5 篇或 4 个，尽量做，最后报告说明

## 6. 阅读文章（最多 5 篇）

<!--
  目的：打开未读文章，模拟用户阅读行为（滚动浏览）。
  
  策略：
    - 按顺序遍历 unread_articles 列表
    - 每篇打开后，分 8 段滚动，每段停留 2 秒，模拟真实阅读
    - 总停留时间约 20 秒/篇
  
  终止条件（满足任一即停止）：
    - 成功读完 5 篇
    - 连续 3 篇失败（导航超时或页面出错）
    - 列表已读完
  
  成功判定：页面 title 非空且不含 404/error/not found
-->

对 `unread_articles` 依次执行：

1. 导航到 `https://community.arc.network/home/${article.slug}`（15 秒超时）
2. 模拟阅读（滚动浏览）：
   ```js
   async () => {
     await new Promise(r => setTimeout(r, 2000)); // 等页面加载
     const h = document.body.scrollHeight;
     // 分 8 段平滑滚动，模拟人眼浏览
     for (let i = 1; i <= 8; i++) {
       window.scrollTo({ top: (h * i) / 9, behavior: 'smooth' });
       await new Promise(r => setTimeout(r, 2000)); // 每段停留 2 秒
     }
     await new Promise(r => setTimeout(r, 2000)); // 末尾多停留一会
     return { title: document.title };
   }
   ```
3. 标题非空且不含 404/error → 加入 `success_articles`

## 7. 观看视频（最多 4 个）

<!--
  目的：打开未观看的视频，模拟点击播放并等待播放。
  
  关键实现：
    - Arc 使用 Wistia 或 YouTube 嵌入视频
    - 优先通过 take_snapshot 找 "Play Video" / "Play video" 按钮
    - 找不到按钮时，通过 CDP 真实鼠标点击 iframe 中心位置
      （iframe.click() 受跨域限制无法触发播放）
    - 点击后等待 40 秒确保播放时间被记录
  
  注意：不同视频平台的按钮文字不同：
    - Wistia：大写 V "Play Video"
    - YouTube：小写 v "Play video"
-->

对 `unwatch_videos` 依次尝试，最多 4 个：

1. 导航到视频页 `https://community.arc.network/home/${video.slug}`
2. 等待页面加载（3~5 秒）
3. 找播放按钮：
   - 优先通过 take_snapshot 找 `role=button` 且文本匹配 `Play [Vv]ideo`
   - 没找到则用 JS 点击 Wistia iframe
   ```js
   () => {
     const f = document.querySelector('iframe[src*="wistia"], iframe[src*="player"], iframe[allow*="autoplay"]');
     if (f) { f.click(); return 'clicked-iframe'; }
     return 'not-found';
   }
   ```
4. 播放后等待 **40 秒**（确保计分系统记录）
5. 成功 → 加入 `success_videos`，继续播放下一个

## 8. 生成报告

<!--
  目的：汇总本次执行结果，展示给用户。
  格式示例见下方模板。
-->

输出格式：
```
执行完成！
- 成功阅读 X/5 篇：1. <title> 2. <title> ...
- 观看视频 Y/4 个：1. <title> 2. <title> ...
- 预计积分：{X*2 + Y*4} 分
- 剩余库存：文章 N 篇 / 视频 N 个
- 5 分钟后到账，到 my-contributions 页面确认
```

---

## 积分规则备忘录

<!--
  实际测试得出的积分规则（随时可能变化）：
-->

| 操作 | 每单位积分 | 每日上限 | 说明 |
|------|:---------:|:--------:|------|
| 📖 阅读博客 | +2 | 5 篇 = +10 | 同一篇 24h 内重复读不给分 |
| 📺 观看视频 | +4 | 4 个 = +16 | 同上，需点击播放 |
| 📅 每日签到 | +1 | 1 次 = +1 | 访问首页后自动触发 |
| ❌ External Content | 0 | - | 实测不计分，已过滤 |
| ❌ Resource | ? | - | 部分 Resource 可能不计分 |

## 常见问题

<!--
  使用过程中遇到的坑和解决方法：
-->

| 问题 | 原因 | 解决 |
|------|------|------|
| 阅读不给分 | 同一篇文章 24h 内重复读 | 选从未读过的文章 |
| 视频不给分 | iframe.click() 跨域无效 | 用 CDP 真实鼠标点击 iframe 中心 |
| 找不到播放按钮 | Wistia/YouTube 按钮文字不同 | 同时匹配 "Play Video" 和 "Play video" |
| 积分不到账 | 需要等待 | 约 5 分钟后到账 |
| MCP 连不上 | 调试 Chrome 没启动 | 运行 launch-chrome-debug.bat |

## 错误处理

<!--
  各环节异常时的兜底策略：
-->

| 错误 | 处理方式 |
|------|---------|
| MCP 连接失败 | 提示用户启动调试 Chrome |
| 未登录 | 停止，等待用户手动登录 |
| 文章导航超时/404 | 跳过，连续 3 次失败终止阅读 |
| 视频无法播放 | 换下一个，最多试 3 个 |
| 库存不足 | 能做多少做多少 |
