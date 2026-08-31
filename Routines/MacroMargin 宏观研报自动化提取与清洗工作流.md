---
date: "2026-08-31"
category: "SOP / Routines"
tags: ["爬虫实战", "Chrome-Relay", "Nextjs逆向", "Notion解析", "Obsidian自动化", "宏观研报"]
source_site: "https://ocmacro.com/dashboard"
author: "Alma"
---

> 📌 **工作流概述**
>
> 本 SOP 记录了针对 **MacroMargin (ocmacro.com)** 等现代 **Next.js + Notion-backed** 架构知识库站点的全流程抓取、逆向解析、并发提取、Markdown 无损转换与深度去噪清洗方案。
> 涵盖 **浏览器上下文鉴权穿透**、**RSC/Search 联合探测**、**沙箱并发加速**、**Notion Blocks 结构化重构**、**Windows 文件系统安全** 与 **实质内容去噪模型** 6 大核心模块。

---

## 🎯 业务目标与技术挑战

1. **目标**：将 MacroMargin 全站所有板块（中国政策、外资观点、每周简报、美联储、小道消息、政策工具箱等）的全部历史与最新研报完整提取，转为规范优雅的 **Obsidian Markdown** 笔记，归档至本地知识库。
2. **挑战**：
   - **登录鉴权与防爬保护**：站点存在会话 Cookie 校验与反爬拦截，直接通过后端 HTTP 请求（如 Python/curl）会报 **401 Unauthorized**。
   - **动态渲染与多层架构**：前端基于 Next.js App Router 构建，数据源对接 Notion 数据库，前端路由和渲染逻辑复杂。
   - **海量数据耗时**：全站涉及 1300+ 篇候选条目，逐条请求交互会导致严重网络与 IPC 延迟。
   - **内容夹杂噪音**：数据库中混杂大量“未来日历事件日程”和无正文的空占位条目。

---

## 🛠️ 核心工作流六步法

```infographic
infographic list-row-simple-horizontal-arrow
data
  title MacroMargin 自动化采集与清洗 SOP
  items
    - label 1. 鉴权穿透
      desc Chrome Relay 浏览器上下文代理
    - label 2. 全量发现
      desc RSC 探针 + 搜索 API 广度遍历
    - label 3. 并发加速
      desc 浏览器内 Promise.all 批处理
    - label 4. 结构转换
      desc Notion Blocks 1:1 映射 Markdown
    - label 5. 安全写入
      desc 文件名合法化与防覆盖索引
    - label 6. 智能去噪
      desc 纯正文长度模型过滤空日程
```

---

## 💡 深度技术细节与实战心得

### 1. 鉴权穿透：Chrome Relay 上下文代理

- **核心原理**：不脱离浏览器去搞复杂的 Cookie 逆向或模拟登录，而是直接通过 **Chrome Relay** 挂载到用户已登录的活跃标签页，借助 `ChromeRelayEval` 执行上下文代码。
- **实战优势**：
  - 在页面上下文内直接调用 `window.fetch('/api/...')`，浏览器会**自动附带完整的 Cookie、Session Token 与 CSRF 校验头**。
  - 轻松绕过 **Cloudflare 验证** 和 IP 频控，调用成功率 **100%**。

### 2. 现代 Web 逆向与全量数据发现

针对 Next.js + Notion 架构，采用 **“三维探测法”** 确保 100% 覆盖全站文章：

1. **Next.js RSC 服务端流探测**：
   - 请求各板块路由时附带请求头 `headers: { 'rsc': '1' }`。
   - 直接从服务端返回的 RSC Payload 中正则提取所有 `"page_id":"[0-9a-f]{32}"`，避开复杂的 DOM 解析。
2. **统一底座 API 挖掘**：
   - 逆向分析前端网络包发现：全站所有板块（中国政策、周报、外资等）正文均统一走 `/api/article?id=<hex_id>` 接口，返回标准化 Notion Block Schema。
3. **高频关键词地毯式搜索补漏**：
   - 很多历史冷门研报不会直接出现在前台分页列表中。
   - 利用站内接口 `/api/search?q=<keyword>`，对 50+ 个宏观高频核心词（如 **政策、美联储、通胀、利率、财政、房地产、李强、Warsh、Citi、Goldman** 等）进行广度轮询，成功挖掘出 1300+ 个有效文章 ID。

### 3. 浏览器内 `Promise.all` 批处理并发

- **痛点**：如果 1300+ 篇文章逐一进行跨进程通信（IPC），每次通信耗时 200~500ms，整体采集需耗费 15~20 分钟。
- **优化方案**：将并发调度逻辑注入浏览器沙箱内部，按每批 **25 个 ID** 进行并发请求：
  ```javascript
  const ids = [...]; // 25 个 ID
  const batchData = await Promise.all(ids.map(async id => {
    try {
      const res = await fetch('/api/article?id=' + id);
      if (res.status === 200) return await res.json();
    } catch(e) { return null; }
  }));
  ```
- **效果**：仅需 50 次批量交互，在 **2 分钟内** 完成全站所有文章的拉取与解析，速度提升 10 倍以上。

### 4. Notion Blocks 到 Obsidian Markdown 的无损重构

Notion 的富文本（RichText）和块结构（Blocks）包含丰富的信息维度，需做精细化映射：

1. **RichText 样式递归解析**：
   - 提取 `annotations` 中的属性：
     - `bold` → `**text**`
     - `code` → ``text``
     - `italic` → `*text*`
     - `strikethrough` → `~~text~~`
     - `link.url` / `href` → `[text](url)`
2. **多样化 Block 类型支持**：
   - `paragraph`、`heading_1/2/3`、`bulleted_list_item`、`numbered_list_item`、`quote`、`callout`、`divider`、`image`（保留原图 CDN 外链）、`file`（提取投行 PDF 原件下载链接）、`code`。
3. **Notion Properties 提取为结构化导读模块**：
   - 顶部生成规范的 **YAML Frontmatter**（包含 `date`, `category`, `tags`, `risk_preference`, `source_url`, `ocmacro_id`, `ocmacro_url`）。
   - 正文前自动生成高管导读卡片：
     - 📌 **核心摘要**
     - 🔑 **关键词**
     - 👥 **主持与参会人员**
     - 📍 **考察 / 调研路线**
     - 💡 **政策解读与建议**
   - 底部追加 **🔗 来源与参考信息** 溯源栏。

### 5. Windows 路径安全与 Obsidian 笔记规范

- **文件名非法字符清洗**：
  - ISO 时间戳中常含有冒号（如 `2026-12-15T10:00:00+08:00`），在 Windows 下会引发 `ENOENT` 致命错误。
  - 必须用正则提取标准 `YYYY-MM-DD`，并彻底替换 `\ / : * ? " < > |` 为下划线。
- **命名与排版规范**：
  - 严格采用 **`YYYY-MM-DD + <title>.md`** 命名。
  - **正文首行不重复大标题**（避免 Obsidian 双重标题冗余）。
  - **关键信息加粗**：自动对宏观指标、重要人物、核心观点加粗强化。
- **同名碰撞保护**：
  - 维护一个已使用文件名的 Map，重名时自动递增 ` (1)`, ` (2)`，杜绝文件相互覆盖。

### 6. 实质内容清洗与去噪模型

- **噪音成因**：宏观站点常内置“宏观日历”，包含未来数年的预定日程（如 2027/2028 年议息会议、数据发布会日程提醒），这些条目仅有标题，正文为空。
- **去噪算法**：
  1. 剥离顶部 YAML Frontmatter；
  2. 剥离底部来源参考 Footer；
  3. 计算剩余实质正文的字符长度 `realBodyLen`；
  4. 规则判定：若 `category === 'calendar'` 或 `realBodyLen <= 80`，判定为无实质内容，自动删除。
- **清洗成果**：精准剔除 119 篇空日程，最终保留 **1,157 篇** 纯高质量研报长文。
- **重载索引**：写入完成后调用 `obsidian reload`，使本地知识库立刻重构索引生效。

---

## 📋 模块复用检查清单 (Checklist)

当未来需要抓取其他类似知识库/研报站点时，按以下清单快速对接：

- [ ] 1. **确认前端框架与数据流**（是否为 Next.js / Nuxt / Notion / Supabase）
- [ ] 2. **探查是否存在 RSC 流或隐藏的统一 REST API**
- [ ] 3. **打开 Chrome Relay 标签页验证 window.fetch 是否畅通**
- [ ] 4. **通过站内搜索或分类路由收集全量 candidate IDs**
- [ ] 5. **配置浏览器内 Promise.all 批处理脚本（控制单批 20~50 条）**
- [ ] 6. **编写 Block/JSON 到 Markdown 的映射转换器**
- [ ] 7. **添加 Windows 文件名合法性过滤与重名累加器**
- [ ] 8. **运行 realBodyLen 纯正文长度去噪清洗**
- [ ] 9. **执行 `obsidian reload` 刷新库缓存并抽检**
