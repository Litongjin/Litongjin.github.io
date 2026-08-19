---
title: "工作台日报 · 2026-08-19"
date: 2026-08-19 15:04:30
categories: [日报]
tags: [日报]
author: Litongjin
---

# 工作台日报 · 2026-08-19

> 自动生成于 2026-08-19 15:04 · 个人工作台 Agent

## 🔥 行业热点

- [AI usage patterns in software teams](https://linear.app/data) — *Hacker News*
- [The Amazon tax](https://seths.blog/2026/08/the-amazon-tax/) — *Hacker News*
- [Universal health coverage could save $1T and 114k lives a year: study](https://ysph.yale.edu/news-article/universal-health-coverage-could-save-one-trillion-dollars-and-114000-lives-every-year/) — *Hacker News*
- [Memory prices climb 500% in 12 months](https://www.tomshardware.com/pc-components/ram/memory-prices-climb-500-percent-in-12-months-up-to-10x-the-lowest-ever-tracked-prices-128gb-of-ddr5-now-usd3-399) — *Hacker News*
- [Cursor launches Origin, GitHub alternative](https://cursor.com/changelog/origin-code-hosting) — *Hacker News*
- [Being ambitious and being a dad](https://nicholascharriere.com/blog/being-ambitious-and-being-a-dad/) — *Hacker News*
- [OpenLogi](https://openlogi.org/en) — *Hacker News*
- [Sticky wage norms and the real wage cost of unexpected inflation](https://bfi.uchicago.edu/wp-content/uploads/2026/08/BFI_WP_2026-108-1.pdf) — *Hacker News*
- [How does IKEA come up with names for its products?](https://www.ikea.com/se/en/customer-service/knowledge/articles/6f564c4d-2ccc-46de-b643-545a3948dc79.html) — *Hacker News*
- [And then the men with guns tell you to do it anyway](https://shkspr.mobi/blog/2026/08/and-then-the-men-with-guns-tell-you-to-do-it-anyway/) — *Hacker News*

## 🚀 技能提升点（工作总结汇总）

### 1. Quill 内容同步 getHTML
- **技能点**：掌握 vue-quill 中 Delta 与 HTML 的区分，正确选择 API 同步 v-model。
- **坑点**：watch 中用 getContents() 得到 Delta，trim 后把 Delta JSON 当 HTML 用 DOMParser 解析，污染 modelValue。
- **解决方案**：用 quill.getHTML() 或 quill.root.innerHTML 获取真实 HTML 字符串再同步。
- **拓展**：其他富文本编辑器同样要区分数据模型与展示 HTML。
- *来源：admin-workspace 2026-08-12*

### 2. Vue watch 双向同步防循环
- **技能点**：理解 Vue 响应式循环触发机制，掌握写回前值比较的防护方法。
- **坑点**：双向 watch 中数组引用变化 + deep watch 互相唤醒，即使值未变也无限递归。
- **解决方案**：每一步写回前比较新旧值，无变化直接 return；避免深层 watch 无意义触发。
- **拓展**：任何 A→B→A 的状态同步都要加守卫，可抽公共工具。
- *来源：admin-workspace 2026-08-11*

### 3. element-plus 按需组件 CSS 缺失
- **技能点**：识别按需注册组件不等于加载样式，能快速定位并显式 import 所需 CSS。
- **坑点**：unplugin-vue-components + ElementPlusResolver 不会自动补齐所有组件样式，新组件如 el-splitter 退化为块级布局。
- **解决方案**：显式 `import "element-plus/theme-chalk/el-splitter.css"` 等；不要依赖全量 index.css。
- **拓展**：其他组件库按需引入同样要检查样式加载，写组件前先确认 CSS 是否引入。
- *来源：admin-workspace MEMORY.md*

### 4. 公式 latex 的 < > 实体化
- **技能点**：掌握 HTML 解析器对裸 `<` 的处理，养成在属性/内文中安全存储 latex 的规范。
- **坑点**：裸 `<`/`>` 塞进 data-formula 属性会被标签解析截断，或 innerHTML 渲染时吞并后续节点；字面匹配也失配。
- **解决方案**：存储/传输时转义 `&lt;`/`&gt;`；解析 HTML 标签时做引号感知扫描，渲染前对公式内容先转义。
- **拓展**：可沉淀为 latex 安全处理工具函数，统一在渲染/解析两侧兜底。
- *来源：admin-workspace 2026-08-12*

### 5. 跨版本克隆文件防覆盖
- **技能点**：建立“先看目标文件已有结构”的代码合并意识，避免用旧内容整体覆盖新语义文件。
- **坑点**：直接用旧版 type.ts 覆盖新版完整类型声明，导致所有引用类型找不到。
- **解决方案**：保留目标文件原内容，仅追加所需旧代码；用 git show 对照原始版本；lint 校验。
- **拓展**：任何“克隆”操作都应视为合并而非替换，可先 diff 再动手。
- *来源：admin-workspace 2026-08-12*

### 6. 文本匹配容空白
- **技能点**：掌握字面匹配失败时用空白容错正则重试的策略，提升匹配鲁棒性。
- **坑点**：AI 生成的 typo original 与正文 latex 空格不一致，严格字面匹配永远失配。
- **解决方案**：构建 `\s*` 容空白正则，严格匹配优先，失败再容错重试；纯空白字符串返回 null 防死循环。
- **拓展**：可推广到搜索/高亮/替换场景，做成通用工具函数。
- *来源：admin-workspace 2026-08-12*

