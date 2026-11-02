---
title: "工作台日报 · 2026-11-03"
date: 2026-11-03 07:01:23
categories: [工作日记]
tags: ["日报", "大模型", "开源项目", "AI Agent", "AI安全"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-03

## 🔥 行业热点

- [Last night an AI agent performed brain surgery on a mouse](https://twitter.com/OcanaSantero/status/2115442964574908902) — *Hacker News*
  - 📌 **内容**：AI Agent 与机器人系统结合，进入需要高精度闭环控制的医疗操作场景，显示出 agent 从纯软件任务走向物理世界的趋势。
  - 💡 **学习**：可关注 agent 在“感知—规划—执行”闭环中的架构设计，以及如何用工具调用串联机械与视觉模块。
  - 🧭 **拓展**：查阅该项目的论文或开源代码，观察 agent 控制机械臂时的接口与安全约束如何定义。
- [Agentic backdoors via pretraining poisoning](https://arxiv.org/abs/2610.37741) — *Hacker News*
  - 📌 **内容**：预训练数据投毒可在模型中植入难以察觉、由特定触发条件激活的后门行为，对具备工具调用能力的 agent 尤具风险。
  - 💡 **学习**：理解模型供应链安全：数据来源审计、去重清洗与后门检测是训练流程中不可省略的环节。
  - 🧭 **拓展**：可用小模型与公开的后门基准复现投毒实验，验证不同清洗策略的防御效果。
- [Launch HN: Momentic (YC W24) – Agents that catch bugs by using your app](https://momentic.ai/) — *Hacker News*
  - 📌 **内容**：用 AI agent 像真实用户一样操作应用来发现缺陷，试图替代易失效的脚本化端到端 UI 测试。
  - 💡 **学习**：可学习 agent 驱动 E2E 测试的思路：如何生成操作路径、判断页面状态、以及在界面变化时自愈。
  - 🧭 **拓展**：在自己的项目中用 Playwright 结合 LLM 做一个最小的自愈式测试原型进行验证。
- [LongCat-2.5: Native Multimodality for Agentic Tasks](https://longcat.ai/blog/longcat-2.5/) — *Hacker News*
  - 📌 **内容**：面向 agent 任务设计的原生多模态模型，强调文本与视觉等模态在统一架构下协同完成操作类任务。
  - 💡 **学习**：关注多模态模型在工具调用、界面操作等 agent 场景的评测维度与失败模式。
  - 🧭 **拓展**：可下载模型在本地跑 GUI/文档理解类任务，对比纯文本模型的差异。
- [C64 Basic Dungeon Crawler: Stairs, Levels and Levelling Up (Part 11)](https://retrogamecoders.com/c64-basic-dungeon-part11/) — *Hacker News*
  - 📌 **内容**：在 Commodore 64 的 BASIC 环境下逐步构建地牢探索游戏的教学系列，本部分讲解楼梯、关卡与升级机制。
  - 💡 **学习**：从受限平台学起，能更清晰地体会游戏主循环、状态机与关卡数据结构的本质设计。
  - 🧭 **拓展**：在 VICE 等 C64 模拟器中照着系列文章敲代码并扩展自己的关卡逻辑。
- [HackerTube: YouTube built from videos recommended by HN](https://karpathy.bearblog.dev/hacker-tube/) — *Hacker News*
  - 📌 **内容**：把 Hacker News 上被推荐的视频聚合成类似 YouTube 的浏览站点的个人项目。
  - 💡 **学习**：可参考其数据抓取、去重与内容索引的做法，理解小型聚合类产品的实现路径。
  - 🧭 **拓展**：仿照它做一个自己领域（如播客、论文）的聚合站，验证抓取与排序策略。
- [Show HN: HackerBooks – HackerTube, but for books mentioned on HN](https://hackerbooks.org) — *Hacker News*
  - 📌 **内容**：与 HackerTube 同源的小项目，聚合 Hacker News 讨论中被提及的书籍并建成可浏览目录。
  - 💡 **学习**：学习如何从非结构化讨论文本中抽取实体（书名）并做规范化与去重。
  - 🧭 **拓展**：可尝试用 LLM 抽取+规则校验的混合方式改进书籍识别准确率。
- [Astra can read and write weights to improve Qwen](https://rikiyat.github.io/blog/weight-space-programming/) — *Hacker News*
  - 📌 **内容**：直接读取与修改模型权重来改进 Qwen 类模型的技术方向，属于权重级模型编辑与优化。
  - 💡 **学习**：了解权重级干预与微调的区别，以及可解释性工具在定位关键参数上的作用。
  - 🧭 **拓展**：在小型开源模型上复现权重编辑流程，对比微调在效果与成本上的差异。
- [Postgres 19 is late because so many more bugs are being found now](http://rhaas.blogspot.com/2026/10/postgres-19-is-late-and-its-ais-fault.html) — *Hacker News*
  - 📌 **内容**：更完善的测试与自动化手段让更多缺陷被提前发现，反而拉长了成熟数据库项目的发布周期。
  - 💡 **学习**：可体会成熟开源项目中质量门禁、测试覆盖与发布节奏之间的权衡。
  - 🧭 **拓展**：查看该项目的 buildfarm/CI 配置，思考如何把类似测试策略引入自己的项目。
- [Show HN: Subvocal – Talk to your computer without making noise](https://news.ycombinator.com/item?id=50340088) — *Hacker News*
  - 📌 **内容**：通过捕捉默读时的微弱肌肉或生理信号实现无声语音输入，探索新的人机交互输入方式。
  - 💡 **学习**：可了解生物信号采集与语音识别模型结合的工程链路，以及低信噪比数据的处理思路。
  - 🧭 **拓展**：关注开源 EMG/肌电采集方案，评估自己搭建原型所需硬件与信号处理步骤。

## 🌟 GitHub 热门开源项目

- [Graphify-Labs/graphify（Agent Skills）](https://github.com/Graphify-Labs/graphify) — *GitHub · Python · +17.8k/周 · 总 141.2k star*
  - 📌 **是什么**：把一个代码库连同文档、SQL schema、配置和 PDF 一起解析成可查询知识图谱的工具，并以 /graphify 技能形式接入 Claude Code、Cursor、Codex、Gemini CLI 等编码代理。
  - 💡 **学习点**：可以学本地确定性 AST 解析与 GraphRAG 结合的做法，理解如何把非结构化仓库转成 LLM 可检索的结构化上下文。
  - 🧭 **上手**：先读 README 的技能安装与输出格式说明，再在自己的一个前端仓库上跑一次 /graphify，观察生成的节点与边结构。
- [harry0703/MoneyPrinterTurbo（LLM 应用）](https://github.com/harry0703/MoneyPrinterTurbo) — *GitHub · Python · +9.4k/周 · 总 137.5k star*
  - 📌 **是什么**：利用 AI 大模型和自动化工作流，根据主题或关键词一键生成高清短视频。Generate HD short videos from a topic or keyword with an automated AI workflow.
- [NousResearch/hermes-agent（Agent 框架）](https://github.com/NousResearch/hermes-agent) — *GitHub · Python · +9.1k/周 · 总 259.8k star*
  - 📌 **是什么**：一个强调可长期演进与个性化陪伴的通用 AI Agent 项目，围绕工具调用与代理循环构建。
  - 💡 **学习点**：适合观察一个通用 agent 的会话循环、工具注册与记忆管理是如何分层的。
  - 🧭 **上手**：先跑最小启动示例，再定位 agent 主循环文件，画出「输入→规划→工具调用→回填」的流程。
- [JuliusBrussee/caveman（Agent Skills）](https://github.com/JuliusBrussee/caveman) — *GitHub · Go · +7.9k/周 · 总 116.9k star*
  - 📌 **是什么**：一个以「极简说话风格」压缩提示词的技能加代理，用更少的 token 完成同样的编码任务。
  - 💡 **学习点**：可以学提示词工程里的 token 成本控制思路，以及代理层如何拦截并改写请求体。
  - 🧭 **上手**：对比开启与关闭该代理时同一任务的请求 payload，重点看代理转发与提示改写的那段代码。
- [bojieli/ai-agent-book](https://github.com/bojieli/ai-agent-book) — *GitHub · Python · +4.9k/周 · 总 57.0k star*
  - 📌 **是什么**：《深入理解 AI Agent：设计原理与工程实践》的开源主仓库，含全书正文、PDF 与按章配套代码。
  - 💡 **学习点**：适合系统性补齐 agent 设计原理、上下文工程与多智能体等概念的完整知识框架。
  - 🧭 **上手**：先读上下文工程与记忆相关章节，再跑对应章节的配套代码验证书中结论。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步防死循环
- **技能点**：能识别“状态 A→数组→状态 B→状态 A”这类双向 watch 的结构性死循环，并在每一步设置值比较守卫。
- **坑点**：标志位变化触发 watch 写入新数组引用，deep watch 又把值写回标志位；即使最终值没变，双方仍互相唤醒，直到报 “Maximum recursive updates exceeded”。
- **解决方案**：在写回前先比较推导值与当前值，相等就直接 return，不再赋新引用；写前同样逐项判断当前值是否已等于推导值。
```text
const syncSizesFromFlags = () => {
  const next = [leftWidth.value, 'auto', rightWidth.value]
  if (next.every((v, i) => v === localPanelSizes.value[i])) return
  localPanelSizes.value = next
}
watch(localPanelSizes, (v) => {
  const folded = v[0] === 0
  if (isFilePreviewFolded.value !== folded) isFilePreviewFolded.value = folded
}, { deep: true })
```
- **拓展**：双向同步更适合收敛为单一数据源（如只保留 sizes），另一侧用 computed 派生，从根上消除回写环。
- *来源：admin-workspace-new 2026-08-11*

### 2. 事件冒泡定位与非侵入修复
- **技能点**：能从“点击某处引发无关区域展开或收起”反推冒泡链路，并把修复放在最小影响面，而不是改动共享组件。
- **坑点**：footer 里 CopyText 的点击冒泡到外层容器的 @click，触发整卡展开/收起；若在通用复制组件里加 stopPropagation，会改掉所有调用方的行为。
- **解决方案**：在业务调用点加 @click.stop，共享组件回退保持通用（不接收 event、不做 stopPropagation）。
```text
<!-- 共享组件保持通用，阻止冒泡放在业务调用点 -->
<CopyText :text="`题目编号: ${data.questionId}`" :copyText="data.questionId" @click.stop />
```
- **拓展**：通用组件对外只暴露语义，不替调用方决定事件传播策略；修复成本高的项目可在 Code Review 清单里加一条“冒泡类 bug 优先业务侧修”。
- *来源：admin-workspace-test 2026-08-11*

### 3. 富文本工具栏配置化注册
- **技能点**：把“可选按钮清单 + 处理器注册 + 类型约束”收敛到统一注册表，业务侧只按名字声明，不再靠猜 customButtons。
- **坑点**：业务组件不知道 :toolbar-order 能传哪些名字，公式、填空等按钮必须先手配 customButtons 才生效，配置分散且易漏。
- **解决方案**：补全默认按钮映射并让内置 handler 无条件注册，导出 TOOLBAR_BUTTONS 常量与联合类型，业务用 B.xxx 常量声明。
```text
export const TOOLBAR_BUTTONS = ['undo','redo','divider','fillInInsert','formula','image'] as const
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[number]
// type.ts
toolbarOrder?: (ToolbarButtonName | string)[]
```
- **拓展**：注册表可继续扩展为“按钮元数据 + 能力开关”的单一来源，同时驱动文档和 Storybook 示例。
- *来源：admin-workspace-new 2026-08-11*

### 4. 组件能力保留、调用方收窄
- **技能点**：区分“组件通用能力”与“当前业务是否使用”，做到删功能时只删调用方配置，不污染共享组件。
- **坑点**：尺寸记忆、持久化和折叠标志双向同步挤在业务页里，多源状态互相打架；改用 sizes 里的 "hidden" 字符串切显隐更让状态语义含混。
- **解决方案**：删掉业务侧的 storageKey 传参、折叠标志与同步 watch，保留纯拖拽状态；显隐改用 v-if 由 isAiReviewer 直接控制；组件自身的 storageKey 在空 key 时自动跳过，无需改动。
```text
<!-- 调用方不传 storageKey 即关闭持久化；显隐交给 v-if，不再往 sizes 里塞 "hidden" -->
<template v-if="isAiReviewer" #panel-2> ... </template>
```
- **拓展**：把“何时该动共享组件、何时只动调用方”写成决策条目，避免每次重构都扩散改动面。
- *来源：admin-workspace-new 2026-08-11*

### 5. 改完回读磁盘校验落盘
- **技能点**：在多轮/长会话里养成“改完立即回读文件确认”的习惯，识别会话上下文与磁盘不一致的情况。
- **坑点**：会话切换后，前期对文件的 edit 实际没落盘，磁盘仍是旧版；继续基于记忆中的“已改”推进就会漏改或重复改。
- **解决方案**：重新按目标文件当前内容核对并重新应用改动；对已删符号做全引用 grep，确认 0 残留。
```text
grep -rn "TOOLBAR_BUTTONS\|syncSizesFromFlags" src/   # 改完回读磁盘 + 已删符号 0 残留校验
```
- **拓展**：同一文件的多个编辑逐条顺序执行（并行编辑基于同一快照会互相覆盖），收尾统一跑一次残留引用扫描。
- *来源：admin-workspace-new 2026-08-11*

### 6. 存量 lint 与类型分档收严
- **技能点**：掌握大规模存量问题“分档治理”的方法：可机械化的批量脚本 + 命中数断言，需牵链的逐条评估。
- **坑点**：一次性全量收严会拖出签名链和连带文件，改动面爆炸、回归风险高；且规则未清零就升 error 会长期卡住流水线。
- **解决方案**：先分 A（单行可独立替换）/B（牵出签名链）两类，A 类用精确替换脚本批量落地并断言命中数，B 类逐条评估连带修改；升级口径=该规则违规清零才升 error。
```text
// A 类：单行可独立替换 → 批量替换脚本 + 命中数断言
// B 类：牵出签名链 → 逐条评估、连带文件一起改
```
- **拓展**：可沉淀为团队 ESLint/TS strict 升级 checklist：先清存量、再逐条升 error、最后进 CI 门禁。
- *来源：admin-workspace-hr-talent 2026-10-15~10-28*

### 7. 工程知识的分层沉淀
- **技能点**：建立“日报流水 → 领域文档 → 硬约束索引”的三层记忆体系，让排坑结论可检索、可复用、不误导。
- **坑点**：把试错过程、已被覆盖的旧方案、同日多副本内容一股脑写进权威文档，索引失真；多副本各自维护又导致记忆分裂。
- **解决方案**：日常随手记当天日报，收口时把稳定结论写进对应领域文档，MEMORY.md 只留“每次必须遵守的硬约束 + 按场景查的索引”一行指向；多副本用目录联接共享同一份物理文件，归档目录只读。
```text
.codebuddy/memory/MEMORY.md          # 只放硬约束 + 场景索引
.codebuddy/memory/2026-08-11.md      # 日常流水、试错过程
.codebuddy/skills/<domain>/SKILL.md  # 收口后的稳定结论
```
- **拓展**：定期跑一致性体检脚本（校验联接是否失效、索引是否指向存在的文档），把结论过时项显式标注为已修正。
- *来源：admin-workspace 系列 MEMORY.md 2026-10-30*

