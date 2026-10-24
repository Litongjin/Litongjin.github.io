---
title: "工作台日报 · 2026-10-25"
date: 2026-10-25 07:02:50
categories: [工作日记]
tags: ["日报", "大模型", "网络安全", "AI工具", "AI绘图"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-10-25

## 🔥 行业热点

- [A practical guide to AI image generation, from diffusion to agents](https://danielball.com/notes/ai_image_generation.html) — *Hacker News*
  - 📌 **内容**：一篇从扩散模型原理讲到图像生成 Agent 化应用的实践指南，覆盖 AI 图像生成技术栈的演进路径。
  - 💡 **学习**：可以系统了解扩散模型的工作机制，以及如何把图像生成能力包装成 Agent 可调用的工具链。
  - 🧭 **拓展**：可动手用 Stable Diffusion 或 ComfyUI 搭一个图像生成流水线，再尝试用 Agent 框架编排它。
- [Dessa: Conversation Model trained only on speech](https://block.xyz/dessa) — *Hacker News*
  - 📌 **内容**：Dessa 是一个完全基于语音数据训练的对话模型，不依赖文本转写，探索端到端语音对话的新路线。
  - 💡 **学习**：了解纯语音训练对话模型的思路，对比传统的 ASR+LLM+TTS 级联方案的优劣。
  - 🧭 **拓展**：可关注类似项目（如 Moshi、GPT-4o 语音模式）并本地跑一个端到端语音模型验证延迟与自然度。
- [No signs of AI payoff outside the tech sector](https://www.apollo.com/wealth/insights-news/insights/daily-spark/no-signs-of-ai-payoff-outside-the-tech-sector) — *Hacker News*
  - 📌 **内容**：文章指出 AI 的经济回报目前主要集中在科技行业内部，其他行业尚未看到明显的生产率提升证据。
  - 💡 **学习**：有助于程序员理性评估 AI 工具的真实 ROI，避免在业务中盲目夸大 AI 落地效果。
  - 🧭 **拓展**：可对照自己团队的 AI 提效数据做一次量化复盘，验证投入产出比。
- [Why is AI generated language so hard to understand?](https://arxiv.org/abs/2610.29162) — *Hacker News*
  - 📌 **内容**：探讨 AI 生成文本为何常常读起来拗口难懂，涉及模型训练目标与人类语言习惯的错位问题。
  - 💡 **学习**：了解 LLM 输出风格的成因（如 RLHF 偏好、概率采样），从而在写 prompt 或后处理时改善可读性。
  - 🧭 **拓展**：可以对比不同模型的输出风格差异，并尝试用 prompt 工程或微调让输出更贴近人类写作。
- [Neural chip decodes interactions between brain regions](https://actu.epfl.ch/news/neural-chip-decodes-interactions-between-brain-r-2/) — *Hacker News*
  - 📌 **内容**：一种神经接口芯片能够解码大脑不同区域之间的交互信号，属于脑机接口与神经科技硬件方向的进展。
  - 💡 **学习**：可了解神经信号处理与专用芯片设计的交叉领域，拓展对边缘 AI 硬件的认知。
  - 🧭 **拓展**：可关注脑机接口开源项目（如 OpenBCI）了解信号采集与解码的工程实现。
- [Oai-math-revised: a community hub for revising OpenAI's math write-ups](https://terrytao.wordpress.com/2026/10/24/oai-math-revised-a-community-hub-for-revising-openais-math-write-ups/) — *Hacker News*
  - 📌 **内容**：一个社区驱动的项目，旨在修订和校对 OpenAI 发布的数学相关技术文档与推导。
  - 💡 **学习**：学习如何通过社区协作方式审查和修正 AI 论文/技术报告中的数学内容，提升严谨性。
  - 🧭 **拓展**：可参与该项目或类似的开放评审，锻炼阅读大模型数学推导的能力。
- [Argus – a free, open-source agentic security operations platform – MIT](https://thomasunise.com/argus) — *Hacker News*
  - 📌 **内容**：Argus 是一个 MIT 协议的开源安全运营平台，采用 Agent 化架构来自动化处理安全运营任务。
  - 💡 **学习**：学习如何将 Agent 架构应用到 SOC/安全运营场景，如告警分诊、事件响应自动化。
  - 🧭 **拓展**：可拉取源码本地部署，测试其 Agent 在模拟安全事件上的处置流程。
- [Progressive disclosure in agents to make them 2.3x faster](https://www.skyvern.com/blog/deleting-rag-from-our-web-agent-made-it-2-3x-faster/) — *Hacker News*
  - 📌 **内容**：介绍通过渐进式披露（按需加载工具/上下文）来优化 Agent 性能的技巧，据称可带来约 2.3 倍的速度提升。
  - 💡 **学习**：学习如何控制 Agent 的上下文长度与工具加载策略，降低 token 消耗和推理延迟。
  - 🧭 **拓展**：可在自己的 Agent 项目中实现工具懒加载/分层上下文，并基准测试前后延迟对比。
- [Local Wikipedia: Wikipedia, but only for things a half hour walk from you](https://edjefferson.com/localwiki/) — *Hacker News*
  - 📌 **内容**：一个有趣的项目：只展示你步行半小时范围内的维基百科条目，基于地理位置过滤知识内容。
  - 💡 **学习**：可以学习地理数据（OSM/GeoNames）与 Wikipedia API 的结合用法，做一个位置感知的知识应用。
  - 🧭 **拓展**：可复刻该项目，改用不同半径或数据源（如本地 POI）做出自己的本地化知识产品。
- [Today, we hacked what we'd set out to hack: the iPhone](https://blog.calif.io/p/iphone) — *Hacker News*
  - 📌 **内容**：一篇讲述成功攻破 iPhone 的文章，涉及 iOS 安全研究和漏洞利用的经历。
  - 💡 **学习**：可了解 iOS 攻击面、漏洞挖掘与利用链的基本思路，以及苹果的防护机制。
  - 🧭 **拓展**：可结合 Apple 安全研究设备计划（SRD）和公开 writeup 学习 iOS 漏洞研究方法论。

## 🌟 GitHub 热门开源项目

- [Snailclimb/JavaGuide](https://github.com/Snailclimb/JavaGuide) — *GitHub · JavaScript · +385/周 · 总 159.2k star*
  - 📌 **是什么**：一份 Java 后端面试知识体系指南，覆盖计算机基础、数据库、分布式、高并发、系统设计，并新增了 AI 应用开发相关内容。
  - 💡 **学习点**：转型 AI 开发不必抛弃原有后端功底，可借它的 AI 章节了解 Java 生态（Spring AI、MCP）如何接入大模型。
  - 🧭 **上手**：先读仓库中 AI 应用开发相关章节，对照 Spring AI 官方示例理解 LLM 调用与 MCP 概念。
- [jeecgboot/JeecgBoot（LLM 应用）](https://github.com/jeecgboot/JeecgBoot) — *GitHub · Java · +315/周 · 总 48.3k star*
  - 📌 **是什么**：企业级 AI 低代码平台，支持用自然语言生成前后端代码乃至整个系统，内置 AI 聊天、知识库、流程编排与 MCP 插件能力。
  - 💡 **学习点**：学习低代码平台如何把 LLM、RAG、MCP 插件组合成『一句话生成系统』的完整工程闭环。
  - 🧭 **上手**：跑通其 AI 代码生成 demo，观察 prompt 到代码的生成链路与 langchain4j 集成方式。
- [DietrichGebert/ponytail（Agent Skills）](https://github.com/DietrichGebert/ponytail) — *GitHub · JavaScript · +30.5k/周 · 总 175.5k star*
  - 📌 **是什么**：一组面向 Claude Code 等编程助手的 Agent Skills/规则集，主张用最少的代码解决问题的『资深懒人工程师』哲学。
  - 💡 **学习点**：学习如何编写高质量的 Agent 规则与 prompt，让 AI 编码时遵循 YAGNI 等工程原则。
  - 🧭 **上手**：把它的 skill/规则文件装进自己的 Claude Code 或 Cursor，对比有无规则时 AI 输出代码的差异。
- [affaan-m/ECC（Agent Skills）](https://github.com/affaan-m/ECC) — *GitHub · JavaScript · +21.9k/周 · 总 288.1k star*
  - 📌 **是什么**：面向 Claude Code、Codex、Cursor 等多种 Agent 工具的性能优化系统，提供技能、记忆、安全与研究优先的开发流程。
  - 💡 **学习点**：了解 Agent harness 的工程化优化思路：技能组织、记忆管理与跨工具复用。
  - 🧭 **上手**：浏览其目录结构看 skills 与 memory 模块如何组织，挑一个在自己的编码助手里复现。
- [obra/superpowers（Agent Skills）](https://github.com/obra/superpowers) — *GitHub · Shell · +14.0k/周 · 总 304.6k star*
  - 📌 **是什么**：一套 Agent 技能框架与软件开发方法论，通过子代理驱动开发来规范 AI 参与的完整软件生命周期。
  - 💡 **学习点**：学习如何把 SDLC 各阶段拆成可被 AI Agent 执行的技能与流程，而不只是让 AI 写代码片段。
  - 🧭 **上手**：读其方法论文档并跑一个示例流程，体验子代理分工完成一个需求的过程。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向 watch 死循环守卫
- **技能点**：掌握了在 Vue 中做『状态 A → 数组/对象 → 状态 B』双向同步时的防御写法：每个回写点先做值比较再决定是否写。
- **坑点**：折叠标志与 localPanelSizes 双向 watch 互相触发：一侧写新数组引用触发 deep watch，另一侧又回写标志，即使最终值不变也因引用变化无限递归，报 Maximum recursive updates exceeded。
- **解决方案**：两处加守卫：syncSizesFromFlags 先比对 next 与当前三项全等则直接 return；watch(localPanelSizes) 内先比较当前标志 !== 推导值再回写。
```text
const syncSizesFromFlags = () => {
  const next = [leftWidth, 'auto', rightWidth]
  if (next.every((v, i) => v === localPanelSizes.value[i])) return
  localPanelSizes.value = next
}
watch(localPanelSizes, (sizes) => {
  const folded = deriveFolded(sizes)
  if (isFilePreviewFolded.value !== folded) isFilePreviewFolded.value = folded
}, { deep: true })
```
- **拓展**：任何响应式双向桥接（props↔state、URL↔state）都应默认加等值守卫，数组/对象尤其要比内容而非引用。
- *来源：admin-workspace-new 2026-08-11*

### 2. 共享组件保持通用，业务侧拦截冒泡
- **技能点**：学会区分『改共享组件』与『改业务调用点』的边界：通用组件不带业务特判，事件隔离由调用方负责。
- **坑点**：卡片 footer 的复制按钮点击冒泡到外层容器 @click，误触发解析展开/收起；最初想在共享 CopyText 里加 stopPropagation，会污染所有调用方。
- **解决方案**：回退共享组件改动，在业务组件 questionCard.vue 的调用处加 @click.stop；同时用 copyText prop 区分『展示文本』与『实际复制内容』。
```text
<CopyText :text="`题目编号: ${data.questionId}`" :copyText="data.questionId" @click.stop />
```
- **拓展**：共享组件新增能力尽量走可选 prop/slot 且默认行为不变，副作用类修饰符（.stop/.prevent）留给业务侧。
- *来源：admin-workspace-new 2026-08-11*

### 3. 工具栏按钮常量+类型化注册
- **技能点**：掌握了把『字符串约定式配置』升级为『常量 + 联合类型 + 内置默认注册』的 API 设计手法，提升可发现性与类型安全。
- **坑点**：业务方不知道 toolbar-order 能传哪些按钮名；formula/填空等按钮必须额外配 customButtons 才能用，接入成本高且易漏。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 联合类型，getDefaultButtonConfig 补全映射，buildToolbarHandlers 对 formula/image 无条件注册内置 handler，业务侧用 B.xxx 常量声明。
```text
export const TOOLBAR_BUTTONS = { undo: 'undo', redo: 'redo', divider: 'divider', formula: 'formula', image: 'image' } as const
export type ToolbarButtonName = keyof typeof TOOLBAR_BUTTONS
// type.ts: toolbarOrder?: (ToolbarButtonName | string)[]
```
- **拓展**：凡是『靠魔法字符串驱动』的配置口都可沉淀为常量对象 + 类型导出，配合 import type 零运行时成本。
- *来源：admin-workspace-new 2026-08-11*

### 4. 重构分支合并主干勿用镜像格式化
- **技能点**：获得『全量 prettier 分支如何合回主干』的实战经验：直接 merge 后在主干基线上重做语义改动，比镜像格式化冲突少得多。
- **坑点**：把主干也格式化后再 merge（镜像分支法）反而放大冲突（117 文件/414 块 vs 直接 merge 51/118）：合并 base 是未格式化版本，两侧格式化基于不同代码导致行布局错位，还把『仅我方改』的文件变成『双方都改』。
- **解决方案**：直接 git merge 主干 → 冲突处整体采纳主干版 → npm run format 统一排版 → 按清单 grep 驱动地在主干基线上重做语义成果。
```text
git merge master
# 冲突处整体采纳主干版
npm run format
# 按清单 grep 定位，在主干基线上重做语义改动
```
- **拓展**：大格式化与语义重构应拆成独立提交/阶段，合并策略先小规模试跑再定。
- *来源：admin-workspace-hr 2026-10-14*

### 5. 线上现象≠本地代码先对分支
- **技能点**：建立排查直觉：线上/测试环境行为与本地代码对不上时，先怀疑部署的代码版本，而非逻辑本身。
- **坑点**：手机扫码上传报 No static resource file/getStsToken，实际是 v2 迁移全在未提交工作区，测试环境跑的是 master 旧码，白排查半天业务逻辑。
- **解决方案**：用 git show master:<文件> 直接对比部署分支上的文件内容，确认环境跑的是哪份代码，再决定排查方向。
```text
git show master:src/views/upload/index.vue
```
- **拓展**：多副本/多分支仓库引用既有排坑结论前，也要先确认目标副本里对应文件是否存在。
- *来源：admin-workspace-hr 2026-09-24*

### 6. ESLint 违规清零再升 error
- **技能点**：掌握渐进式 lint 治理方法论：先清零再升级规则级别，react-hooks 类违规有成套既定修复手法。
- **坑点**：存量 warning 不清零就直接升 error 会阻塞全员；preserve-manual-memoization 与 exhaustive-deps 在依赖可变值时互相矛盾，set-state-in-effect 在 fetch/初始化 effect 里高频出现。
- **解决方案**：口径=该规则违规清零才可升 error；immutability 类把变量声明整体上移；fetch 类 effect 换 useAsyncEffect（带自建 cleanup 的除外）；弹窗复位用调用方 key 重挂载+同步改 useState 初始值；依赖矛盾先解构为局部变量。
```text
// preserve-manual-memoization 与 exhaustive-deps 矛盾时：先解构
const { userId } = props
const data = useMemo(() => query(userId), [userId])
```
- **拓展**：清 any 可让子代理先分『单行可独立替换/牵出签名链』两类，只对前者脚本批量落地。
- *来源：admin-workspace-hr-talent 2026-10-19*

