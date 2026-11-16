---
title: "工作台日报 · 2026-11-17"
date: 2026-11-17 07:02:36
categories: [工作日记]
tags: ["日报", "可靠性", "网络安全", "AI安全", "AI评测"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-17

## 🔥 行业热点

- [Interpreting claims that a theorem is correct because it was checked in Lean](https://xenaproject.wordpress.com/2026/11/12/and-the-proof-is-checked-in-lean/) — *Hacker News*
  - 📌 **内容**：讨论把定理在 Lean 中形式化验证后，如何理解“机器已证明”的含义与边界。
  - 💡 **学习**：程序员可了解形式化方法与定理证明器如何提升高可信软件、安全协议验证能力。
  - 🧭 **拓展**：可尝试用 Lean 或 Coq 为简单不变式写形式化证明，并接入 CI 验证。
- [Microsoft blames monster email outage on software defect introduced in July fix](https://www.theregister.com/saas/2026/11/16/microsoft-blames-monster-email-outage-on-software-defect-introduced-in-july-fix/5310026) — *Hacker News*
  - 📌 **内容**：微软大型邮件故障被归因于此前修复引入的缺陷，暴露发布与回滚风险。
  - 💡 **学习**：可学习大型服务变更中的灰度发布、回滚策略和事故复盘方法。
  - 🧭 **拓展**：可在自己的服务里引入变更冻结、渐进发布与故障演练来验证。
- [OpenAI: Strengthening the fight against online child sexual abuse](https://openai.com/index/strengthening-online-child-safety/) — *Hacker News*
  - 📌 **内容**：文章方向是 OpenAI 在内容安全、滥用检测与平台治理方面的举措。
  - 💡 **学习**：了解大模型产品如何构建 AI 安全、内容审核与滥用防护策略。
  - 🧭 **拓展**：可调研多模态或文本审核工具，并评估其在应用层的拦截召回与误报。
- [Spamhaus new email sender requirements in November](https://www.spamhaus.org/sender-guidelines/) — *Hacker News*
  - 📌 **内容**：Spamhaus 收紧邮件发件要求，影响邮件送达、反垃圾与发件信誉实践。
  - 💡 **学习**：了解邮件认证、SPF/DKIM/DMARC 配置和发件信誉管理。
  - 🧭 **拓展**：可在测试域名配置邮件认证策略，并观察送达率与退信报告。
- [First phase of embedded assessments at OpenAI](https://openai.com/index/our-first-phase-of-embedded-assessments/) — *Hacker News*
  - 📌 **内容**：可能涉及 OpenAI 将评估能力内嵌到产品或研发流程的阶段性实践。
  - 💡 **学习**：学习 AI 模型上线前的安全评测、回归评估和指标体系建设。
  - 🧭 **拓展**：可建立模型评测集，把关键能力与风险检查嵌入 CI/CD。
- [Universal XSS in uBlock Origin from website-driven element picker](https://iter.ca/post/ubo-uxss/) — *Hacker News*
  - 📌 **内容**：围绕 uBlock Origin 元素选择器可被网站触发的 XSS/安全漏洞展开讨论。
  - 💡 **学习**：了解浏览器扩展与注入式工具中的跨站脚本、权限边界和输入验证。
  - 🧭 **拓展**：可在受控测试页面复现类似选择器交互，检查 CSP 与事件绑定。
- [Software is breaking down at a systemic level](https://www.breck-mckye.com/blog/2026/11/software-is-breaking-down-at-a-systemic-level/) — *Hacker News*
  - 📌 **内容**：讨论软件系统复杂度上升、可靠性下降以及工程实践面临的结构性问题。
  - 💡 **学习**：可思考可观测性、依赖管理、故障模式和工程纪律在大型系统中的价值。
  - 🧭 **拓展**：可选一个自研服务做依赖图梳理与故障注入实验。
- [The Secret ThinkPad Powerbutton Code to Bring Dead Laptops Back to Life (2007)](https://masnick.com/posts/the-secret-thinkpad-powerbutton-code-to-bring-dead-laptops-back-to-life/) — *Hacker News*
  - 📌 **内容**：回顾 ThinkPad 电源按钮相关的硬件维护与故障恢复技巧。
  - 💡 **学习**：了解笔记本硬件排障、厂商遗留功能和低成本维修知识。
  - 🧭 **拓展**：可在旧设备或维修手册中验证类似电源/恢复组合键和硬件复位流程。
- [Jiff 1.0: A datetime library for Rust](https://burntsushi.net/jiff/) — *Hacker News*
  - 📌 **内容**：介绍 Rust 时间/日期库 Jiff 发布 1.0，面向更清晰可靠的时间处理。
  - 💡 **学习**：学习 Rust 生态中日期时间 API、时区处理与库设计思路。
  - 🧭 **拓展**：可在项目中替换现有 datetime crate，并编写时间解析与换算测试。
- [Carnegie Mellon is putting computer science courses on YouTube for free](https://www.cs.cmu.edu/learn/) — *Hacker News*
  - 📌 **内容**：卡内基梅隆大学把计算机科学课程免费放到 YouTube 上。
  - 💡 **学习**：可获取高校系统课学习路径，补齐算法、系统等计算机基础。
  - 🧭 **拓展**：可选一门课完成作业或项目，并整理成可复用的知识库。

## 🌟 GitHub 热门开源项目

- [tt-a1i/archify（Agent Skills）](https://github.com/tt-a1i/archify) — *GitHub · JavaScript · +12.7k/周 · 总 97.2k star*
  - 📌 **是什么**：一个面向 AI 编码助手的交互式架构图技能，用于把想法、计划或代码结构转成可视化图。
  - 💡 **学习点**：学习如何把编码助手输出转化为可理解、可分享、可迭代的架构可视化能力。
  - 🧭 **上手**：阅读 README 中的安装与调用说明，并选择一个小型前端模块尝试生成架构图。
- [cathrynlavery/diagram-design（Agent Skills）](https://github.com/cathrynlavery/diagram-design) — *GitHub · HTML · +11.2k/周 · 总 67.6k star*
  - 📌 **是什么**：一个面向 AI 编码助手的编辑级图表设计技能仓库，强调自包含 HTML 与 SVG 输出。
  - 💡 **学习点**：学习如何用结构化视觉规范提升图表可读性，而不是只让模型生成粗糙图。
  - 🧭 **上手**：对照示例中的不同图表模板，选择一种架构图替换你的系统边界与数据流。
- [rohitg00/ai-engineering-from-scratch](https://github.com/rohitg00/ai-engineering-from-scratch) — *GitHub · Python · +8.9k/周 · 总 79.2k star*
  - 📌 **是什么**：从零讲解 AI 工程化实践的教程仓库，覆盖大模型、生成式 AI 与智能体应用的学习路径。
  - 💡 **学习点**：帮助前端工程师建立从模型调用到系统交付的 AI/LLM 工程化知识框架。
  - 🧭 **上手**：先看课程目录，选择一个 LLM 应用相关章节并跑通其中的最小示例。
- [usestrix/strix（开发工具）](https://github.com/usestrix/strix) — *GitHub · Python · +7.0k/周 · 总 76.0k star*
  - 📌 **是什么**：面向应用安全测试的开源 AI 工具，试图自动发现并协助修复应用中的漏洞。
  - 💡 **学习点**：学习如何把 AI agent 与代码审查、安全扫描流程结合成可验证工作流。
  - 🧭 **上手**：阅读 README 中的扫描示例，并在一个本地示例项目上运行一次最小安全测试。
- [Shubhamsaboo/awesome-llm-apps（LLM 应用）](https://github.com/Shubhamsaboo/awesome-llm-apps) — *GitHub · Python · +2.9k/周 · 总 144.9k star*
  - 📌 **是什么**：收录大量 LLM 应用、智能体与 RAG 示例的开源应用集合。
  - 💡 **学习点**：快速理解 AI/LLM 应用常见形态，以及可复用的工程样板。
  - 🧭 **上手**：从 RAG 或 agent 分组中选择一个简单示例启动，追踪用户输入到模型输出的链路。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue双向同步递归更新
- **技能点**：掌握状态派生与 watch 副作用循环识别，能用等值守卫或单向化降低无限递归风险。
- **坑点**：折叠标志与面板尺寸双向同步时，每次生成新数组触发 deep watch，即使最终值不变也会互相唤醒，导致 Maximum recursive updates。
- **解决方案**：写入前逐字段比较守卫；更彻底是去掉双向同步，让拖拽尺寸做单一真源，右栏显隐直接用条件渲染控制。
```text
const same = a => a[0] === sizes.value[0] && a[1] === sizes.value[1] && a[2] === sizes.value[2]
if (same(next)) return
sizes.value = next
watch(sizes, next => { const folded = next[0] === 0; if (folded !== isFolded.value) isFolded.value = folded })
```
- **拓展**：沉淀为状态同步检查清单：优先单向事件/computed，必须双向时加幂等比较和来源标记。
- *来源：admin-workspace-new 2026-08-11*

### 2. 事件冒泡边界控制
- **技能点**：掌握事件隔离的组件边界设计：通用组件不替业务决定冒泡，业务在调用处收口。
- **坑点**：复制按钮点击冒泡到外层卡片导致误触发展开或收起；若直接给共享复制组件加 stopPropagation，会污染通用行为并影响其他场景。
- **解决方案**：业务组件里给共享组件加 @click.stop；共享组件保持无事件副作用，只按 prop 复制稳定 ID。
```text
<CopyText :text="label" :copyText="id" @click.stop />
```
- **拓展**：通用组件优先通过 emit 或事件参数交还控制权，避免硬编码业务场景。
- *来源：admin-workspace-new 2026-08-11*

### 3. 编辑器工具栏API契约
- **技能点**：掌握可发现性组件 API 设计：用类型化常量暴露可选注册名，降低隐性 customButtons 依赖。
- **坑点**：toolbar-order 字符串不透明，公式或图片等功能若未同步注册 handler 与样式，业务容易漏配；编辑未落盘还会造成验证假象。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与联合类型，内置 handler 无条件注册，按钮图标和标题统一补齐，并用 lint 与磁盘 diff 验证落盘。
```text
export const TOOLBAR_BUTTONS = { undo: 'undo', formula: 'formula', image: 'image' } as const
export type ToolbarButtonName = keyof typeof TOOLBAR_BUTTONS
interface Props { toolbarOrder: ToolbarButtonName[] }
```
- **拓展**：复杂插件组件可沉淀选项清单：注册名、handler、样式、示例文档四位一体。
- *来源：admin-workspace-new 2026-08-11*

### 4. React Hook副作用收敛
- **技能点**：掌握用 remount、初始值同步和稳定依赖收敛 Hook 副作用与 memo 约束冲突。
- **坑点**：弹窗打开重置用 effect setState 触发 set-state-in-effect；机械替换 fetch effect 会误伤带自建 cleanup 的异步逻辑；可变对象依赖同时引发 preserve-manual-memoization 与 exhaustive-deps 冲突。
- **解决方案**：复位场景改父组件 key remount 并同步 useState 初始值；初始化类异步用 useAsyncEffect 但保留自建 cleanup 场景；先解构出稳定局部变量再入依赖。
```text
<Dialog key={openCount} initialData={data} isOpen={open} />
const { id } = item
const value = useMemo(() => transform(item), [id])
```
- **拓展**：形成 hooks lint 分档：可机械替换、需语义判断、需业务重构三类，避免一刀切。
- *来源：admin-workspace-hr-talent 2026-10-19*

### 5. 线上现象先查部署分支
- **技能点**：掌握以部署分支代码和文件存在性作为事实基线，区分代码版本、构建产物与运行环境差异。
- **坑点**：本地工作区已有改动但未部署，测试环境仍跑 master 旧码；多 clone 或多分支下直接套用记忆结论，会误判文件不存在或功能缺失。
- **解决方案**：先看 git show origin/master 与目标副本当前分支及文件是否存在，再判断是环境问题、缓存问题还是代码问题。
```text
git show origin/master:src/service/upload.ts
ls src/components/QuillEditorNew
```
- **拓展**：发布校验可固化为：部署分支快照、关键文件存在性、接口前缀与资源路径 diff。
- *来源：admin-workspace-hr 2026-09-24*

### 6. 格式化分支合并策略
- **技能点**：掌握大规模代码格式化与主干合并的冲突控制，能把排版差异和语义成果分开处理。
- **坑点**：把主干也格式化到镜像分支再合并，会使合并 base 与两侧行布局错位，冲突放大；原本只我方改的文件会变成双方都改。
- **解决方案**：直接 merge 主干，冲突处优先采纳主干基线；再统一 format，最后按语义成果清单在主干基线上重放业务改动。
```text
git merge origin/master
npm run format
# then replay checklist items onto merged baseline
```
- **拓展**：将纯格式化独立成小 PR 并锁定基线，合并期间冻结大范围排版改动。
- *来源：admin-workspace-hr 2026-10-14*

