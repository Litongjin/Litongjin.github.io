---
title: "工作台日报 · 2026-11-23"
date: 2026-11-23 07:01:48
categories: [工作日记]
tags: ["日报", "AI Agent", "AI基础设施", "AI安全", "C++"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-23

## 🔥 行业热点

- [Pandex – Official European Law for AI Agents, over MCP](https://www.pandex.io) — *Hacker News*
  - 📌 **内容**：一个将欧洲法律法规以 MCP 可访问形式提供给 AI Agent 的项目，方向是让 Agent 在法律场景下查询、引用和推理。
  - 💡 **学习**：可学习如何把权威法规数据封装成 MCP 工具或知识源，并用于约束 Agent 的合规行为。
  - 🧭 **拓展**：可尝试用本地法规文本搭建 MCP server，验证 Agent 回答时对条文引用的准确性。
- [Show HN: I built a game where you are an LLM's subagent](https://github.com/gulivan/worker755) — *Hacker News*
  - 📌 **内容**：一个让玩家扮演 LLM 子代理的游戏，围绕任务分解、提示词和代理协作展开。
  - 💡 **学习**：可通过游戏化方式理解多 Agent 系统中的子任务拆分、上下文传递和 LLM 工作流设计。
  - 🧭 **拓展**：可用 LangGraph 或 AutoGen 搭一个最小 subagent 工作流来对照体验。
- [A Beginner's Guide to Local AI with Inkling, Qwen, and Unsloth](https://www.gpupartner.com/blog/beginners-guide-local-ai-inkling-qwen-unsloth) — *Hacker News*
  - 📌 **内容**：面向新手的本地 AI 实践指南，涉及 Inkling、Qwen 模型和 Unsloth 工具链的组合使用。
  - 💡 **学习**：可学习本地部署开源模型、使用 Unsloth 加速微调，以及搭建私有 AI 应用流程。
  - 🧭 **拓展**：可在消费级 GPU 上运行 Qwen 小模型，并对比不同量化方案的效果。
- [Muru – AI-assisted Rust server for Python apps by mod_WSGI's creator](https://simonwillison.net/2026/Nov/21/muru/) — *Hacker News*
  - 📌 **内容**：mod_WSGI 作者推出的 Rust 服务器项目，面向 Python 应用并强调 AI 辅助能力。
  - 💡 **学习**：可关注用 Rust 提升 Python 应用服务层性能，以及 AI 辅助服务器工具链的设计思路。
  - 🧭 **拓展**：可将一个 Flask 或 FastAPI 应用放到 Muru 前做性能与部署对比。
- [C++26 is done (again): November 2026 ISO C++ meeting trip report](https://herbsutter.com/2026/11/21/c26-is-done-again-trip-report-november-2026-iso-c-standards-meeting-buzios-brazil/) — *Hacker News*
  - 📌 **内容**：C++26 标准相关会议报告，介绍标准定稿后的进展和对编译器生态的影响。
  - 💡 **学习**：可了解 C++26 的特性方向、标准化流程，以及后续编译器支持节奏。
  - 🧭 **拓展**：可在支持实验特性的编译器中验证新标准特性。
- [Agent swarm trying to solve P vs. NP](https://pequalnp.com/) — *Hacker News*
  - 📌 **内容**：一个多智能体群尝试解决 P vs. NP 问题的实验，体现 AI Agent 在复杂理论问题上的协作探索。
  - 💡 **学习**：可学习多 Agent 任务拆分、长程推理和验证机制如何用于开放性难题。
  - 🧭 **拓展**：可给 Agent 群加入形式化证明检查器来验证输出质量。
- [Harvard CS 2881R AI Safety](https://boazbk.github.io/mltheoryseminar/) — *Hacker News*
  - 📌 **内容**：哈佛 AI Safety 课程信息公开，方向是系统性讲解 AI 风险、对齐与安全研究。
  - 💡 **学习**：可借助课程材料补齐 AI 安全、对齐、评估和红队测试等基础知识。
  - 🧭 **拓展**：可结合公开论文复现一个简单的模型安全评估案例。
- [Lean verification of the Riemann Hypothesis](https://leanprover-community.github.io/blog/posts/riemann-hypothesis/) — *Hacker News*
  - 📌 **内容**：使用 Lean 形式化验证黎曼猜想相关工作的讨论，体现形式化数学与证明辅助工具的交集。
  - 💡 **学习**：可学习 Lean 等定理证明器如何表达复杂数学命题，以及形式化验证的工程约束。
  - 🧭 **拓展**：可用 Lean 写一个小定理证明，体验类型驱动证明流程。
- [Data Processing Incidents on Inference Providers](https://openrouter.ai/blog/announcements/data-processing-incidents-on-inference-providers/) — *Hacker News*
  - 📌 **内容**：讨论推理服务商的数据处理事故，涉及 AI API、日志、隐私和第三方推理平台风险。
  - 💡 **学习**：可学习在调用外部推理服务时如何评估数据边界、脱敏和审计需求。
  - 🧭 **拓展**：可梳理自家 AI 应用调用第三方模型时的数据流和泄露风险点。
- [Open Social Web Alliance](https://osw-alliance.net/en/) — *Hacker News*
  - 📌 **内容**：开放社交网络联盟相关动向，涉及开放协议、跨平台身份和去中心化社交产品。
  - 💡 **学习**：可了解开放社交协议、联邦化架构和平台互操作性的产品设计思路。
  - 🧭 **拓展**：可研究 ActivityPub 或类似协议，搭一个最小联邦社交实例。

## 🌟 GitHub 热门开源项目

> 📈 本周候选 61 个仓库中「Agent 框架」占 22 个，是当前最活跃赛道；周增量破千的仓库：thedotmack/claude-mem、firecrawl/firecrawl、paperclipai/paperclip。

- [thedotmack/claude-mem（Agent Skills）](https://github.com/thedotmack/claude-mem) — *GitHub · TypeScript · +16.7k/周 · 总 119.1k star*
  - 📌 **是什么**：为 Agent 提供跨会话持久上下文，捕获会话过程并用 AI 压缩后注入后续会话。
  - 💡 **学习点**：学习如何设计 Agent 的长期记忆、上下文压缩与相关片段检索注入流程。
  - 🧭 **上手**：先看会话捕获与上下文注入的主入口，再跑一个最小 Claude 会话示例观察记忆回写。
- [firecrawl/firecrawl（开发工具）](https://github.com/firecrawl/firecrawl) — *GitHub · TypeScript · +12.7k/周 · 总 207.0k star*
  - 📌 **是什么**：面向 Agent 的知识获取基础设施，统一搜索、抓取和数据源访问能力。
  - 💡 **学习点**：学习如何把网页抓取、正文提取和 Markdown 输出封装成 Agent 可调用的数据工具。
  - 🧭 **上手**：跑一个 URL 转 Markdown 的最小示例，再对比原始 HTML 与输出结构以理解抽取规则。
- [paperclipai/paperclip（LLM 应用）](https://github.com/paperclipai/paperclip) — *GitHub · TypeScript · +9.8k/周 · 总 116.0k star*
  - 📌 **是什么**：一个用于在工作中管理和编排 Agent 的开源应用，关注任务协作与治理场景。
  - 💡 **学习点**：学习如何把 Agent 的任务、权限、治理和团队协作流程产品化。
  - 🧭 **上手**：从示例任务入口读起，观察它如何注册、调度和查看一个 Agent 任务。
- [mvanhorn/last30days-skill（Agent Skills）](https://github.com/mvanhorn/last30days-skill) — *GitHub · Python · 周增量统计中 · 总 72.7k star*
  - 📌 **是什么**：一个研究型 Agent 技能，从多个平台收集近期信息并生成有依据的摘要。
  - 💡 **学习点**：学习多源检索、信息综合与生成可控摘要的技能封装方式。
  - 🧭 **上手**：选一个主题运行其研究命令，检查引用来源与摘要之间是否可追溯。
- [herdrdev/herdr（开发工具）](https://github.com/herdrdev/herdr) — *GitHub · Rust · +6.9k/周 · 总 52.3k star*
  - 📌 **是什么**：面向编码 Agent 的运行时环境，关注终端复用、进程承载和开发工具链集成。
  - 💡 **学习点**：学习如何为编码 Agent 设计安全执行、进程管理和终端协作等基础设施。
  - 🧭 **上手**：运行 CLI 示例，在同一会话中观察它如何承载一个编码 Agent 的输入输出。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向同步防递归
- **技能点**：掌握响应式双向同步的闭环排查能力，能识别 watch 互触发与引用变化导致的递归更新。
- **坑点**：折叠状态与面板尺寸数组双向 watch，数组新引用加 deep 监听互相唤醒，触发 Maximum recursive updates。
- **解决方案**：每次回写前做等值比较，无变化直接返回；更稳做法是删除双向同步，保留单一状态源。
```text
if (isEqual(next, sizes.value)) return; sizes.value = next;
```
- **拓展**：可沉淀为 useBidirectionalSync 工具，内置浅比较与调试日志。
- *来源：admin-workspace-new 2026-08-11*

### 2. DOM 事件冒泡边界
- **技能点**：能定位嵌套点击冒泡导致的误触发，并明确共享组件与业务组件的职责边界。
- **坑点**：复制按钮点击冒泡到外层卡片点击，意外触发展开/收起。
- **解决方案**：在业务使用处对复制组件加 @click.stop；共享组件保持通用，不内置 stopPropagation。
```text
<CopyText :copyText='id' @click.stop />
```
- **拓展**：对可交互子组件统一提供 stop/propagation 可选项或文档提示。
- *来源：admin-workspace-new 2026-08-11*

### 3. 组件配置契约化
- **技能点**：通过常量、联合类型和默认注册，把隐式可配置项变成可发现、可校验的组件契约。
- **坑点**：业务不知道工具栏可传哪些按钮，部分能力必须额外配 customButtons 才能用。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与类型，补全默认按钮映射并统一注册 handler。
```text
export const TOOLBAR_BUTTONS = ['bold', 'formula', 'image'] as const;
```
- **拓展**：可继续提供运行时校验与文档自动生成。
- *来源：admin-workspace-new 2026-08-11*

### 4. 通用组件插槽透传
- **技能点**：掌握用插槽透传扩展通用组件，既保留内核行为又允许业务定制外观。
- **坑点**：折叠图标需要业务自定义，但直接改通用组件会污染共享逻辑。
- **解决方案**：通用组件提供 #start-collapsible 插槽并隐藏默认图标，业务侧注入 SVG 与状态样式。
```text
slots['start-collapsible'] ? renderSlot() : renderDefault()
```
- **拓展**：可把图标、文案、禁用态都抽成可替换插槽或 render props。
- *来源：admin-workspace 2026-08-10*

### 5. 主干合并格式化策略
- **技能点**：能在长期重构分支合并主干时控制冲突面，避免格式化污染合并基线。
- **坑点**：用本地镜像格式化分支先格式化主干再合并，反而放大冲突，把单侧修改变成双侧修改。
- **解决方案**：直接合并主干，冲突处优先采纳主干版本，再统一格式化并在主干基线上重做语义改动。
```text
git merge master && npm run format && git diff --check
```
- **拓展**：可沉淀合并检查清单：冲突统计、format 后 diff、语义点复核。
- *来源：admin-workspace-hr-talent 2026-10-14*

### 6. 存量 Lint/TS 治理
- **技能点**：能制定渐进式静态检查收严策略，用快速校验和分类清理降低存量风险。
- **坑点**：一次性升 error 或直接全量清 any 容易被存量淹没，且误伤高代价改动。
- **解决方案**：按规则清零后才升级，A 类可批量替换，B 类签名链逐条评估；默认用 tsc/lint 快速验证。
```text
npx tsc --noEmit && npx eslint src --max-warnings=0
```
- **拓展**：可将清零进度做成报表，保留刻意逃生阀清单。
- *来源：admin-workspace-hr-talent 2026-10-19*

