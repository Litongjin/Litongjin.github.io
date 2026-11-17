---
title: "工作台日报 · 2026-11-18"
date: 2026-11-18 07:02:03
categories: [工作日记]
tags: ["日报", "AI工具", "大模型", "前端", "软件工程"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-18

## 🔥 行业热点

- [Harvard researchers find AI coding agents shifted the bottleneck to code review](https://complexdiscovery.com/harvard-researchers-find-ai-coding-agents-shifted-the-bottleneck-to-code-review/) — *Hacker News*
  - 📌 **内容**：讨论 AI 代码生成普及后，团队效率瓶颈可能从编写代码转向人工审查与验证。
  - 💡 **学习**：可学习如何围绕 AI 生成代码建立审查清单、自动化测试和风险控制流程。
  - 🧭 **拓展**：在团队中统计 Agent 生成 PR 的缺陷类型、审查耗时和回滚率。
- [MAI Code 2 and Code 2 Flash](https://microsoft.ai/news/introducing-mai-code-2-and-mai-code-2-flash/) — *Hacker News*
  - 📌 **内容**：标题指向 MAI Code 系列新版本或变体的发布与讨论。
  - 💡 **学习**：可关注其在代码生成、上下文理解、工具调用或速度成本上的定位。
  - 🧭 **拓展**：用相同代码任务对比不同编码模型的输出质量和响应效率。
- [Launch HN: StarSling (YC P25) – Self-improving CI for agent-volume repos](https://starsling.dev) — *Hacker News*
  - 📌 **内容**：面向大量 Agent 提交仓库的自改进持续集成方向，目标是应对 AI 生成变更带来的质量压力。
  - 💡 **学习**：可学习用 CI、测试门禁和自动反馈机制来管理 Agent 驱动的代码仓库。
  - 🧭 **拓展**：为高吞吐仓库增加自动测试、失败聚类和安全回滚策略。
- [Show HN: TrnRT – An open-source LLM inference engine for AWS Trainium](https://github.com/RunanywhereAI/trnrt) — *Hacker News*
  - 📌 **内容**：面向 AWS Trainium 加速芯片的开源大模型推理引擎项目，关注硬件适配与推理性能。
  - 💡 **学习**：可学习 LLM 推理引擎如何围绕专用加速器做调度、批处理和延迟优化。
  - 🧭 **拓展**：在兼容云实例上压测不同模型和批大小下的吞吐、显存占用与成本。
- [Fixing a WebKit bug to see if promises are made of train tracks](https://alistair.sh/thenables) — *Hacker News*
  - 📌 **内容**：通过修复 WebKit 中与 promise 行为相关的 bug，探讨浏览器底层异步执行机制。
  - 💡 **学习**：可学习 JavaScript 事件循环、微任务调度和 promise 在浏览器内核中的实现边界。
  - 🧭 **拓展**：编写最小复现用例，观察不同 promise 解析路径下的回调顺序。
- [GPT‑6.2 Astra and Sol](https://openai.com/index/introducing-gpt-6-2-astra-sol/) — *Hacker News*
  - 📌 **内容**：围绕 GPT‑6.2 变体 Astra 和 Sol 的发布或讨论，重点可能在不同能力定位。
  - 💡 **学习**：可关注不同模型版本在推理、编码、多模态或成本速度上的差异。
  - 🧭 **拓展**：使用固定任务集对不同模型变体做输出质量与稳定性对比。
- [What is the best harness for each model?](https://runta.com/blog/introducing-frontierharness-eval-v1-1/) — *Hacker News*
  - 📌 **内容**：讨论不同模型应如何搭配或配置调用与评测 harness。
  - 💡 **学习**：可学习模型工具链、评测框架和 prompt harness 对结果稳定性的影响。
  - 🧭 **拓展**：为常用模型搭建统一 harness，记录不同配置下的输出差异。
- [How the Claude Code team thinks about verification](https://claude.dev/blog/how-the-claude-code-team-thinks-about-verification/) — *Hacker News*
  - 📌 **内容**：关注 Claude Code 团队如何在 AI 编码流程中理解和设计验证机制。
  - 💡 **学习**：可学习为 AI 生成代码构建测试、静态分析、审查与回滚的闭环。
  - 🧭 **拓展**：在代码生成任务中增加自动测试门禁，并验证不同失败模式的捕获能力。
- [Phrack 73 Released](https://phrack.org/issues/73/introduction_md) — *Hacker News*
  - 📌 **内容**：Phrack 第 73 期发布，延续黑客与安全技术研究社区的交流传统。
  - 💡 **学习**：可从安全刊物中获取漏洞研究、攻击面和系统防御思路。
  - 🧭 **拓展**：在隔离环境中复现当期提到的安全实验或工具用法。
- [jQuery 4.1.0](https://github.com/jquery/jquery/releases/tag/4.1.0) — *Hacker News*
  - 📌 **内容**：jQuery 新版本发布，关注前端库在兼容性、体积和 API 层面的演进。
  - 💡 **学习**：可学习遗留前端库升级策略、DOM 操作行为变化和浏览器兼容回归。
  - 🧭 **拓展**：在测试项目中升级版本并运行自动化回归检查兼容影响。

## 🌟 GitHub 热门开源项目

- [Snailclimb/JavaGuide](https://github.com/Snailclimb/JavaGuide) — *GitHub · JavaScript · +507/周 · 总 159.6k star*
  - 📌 **是什么**：面向 Java 后端与通用面试的知识指南，并把 AI 应用开发纳入工程体系。
  - 💡 **学习点**：适合前端转型者补齐后端工程、分布式与 AI 应用落地所需的基础知识框架。
  - 🧭 **上手**：优先读 AI 应用开发相关章节，再对照 Spring AI/MCP 主题找可运行示例。
- [jeecgboot/JeecgBoot（开发工具）](https://github.com/jeecgboot/JeecgBoot) — *GitHub · Java · +259/周 · 总 48.5k star*
  - 📌 **是什么**：一款开源 AI 低代码平台，覆盖权限、工作流、报表、大屏、AI 知识库等通用能力。
  - 💡 **学习点**：适合前端转型者学习 AI 应用如何与企业级权限、流程和业务代码生成结合。
  - 🧭 **上手**：跑内置 AI 知识库或低代码生成流程，看业务模块如何与 LLM 能力接入。
- [DietrichGebert/ponytail（Agent Skills）](https://github.com/DietrichGebert/ponytail) — *GitHub · JavaScript · +26.8k/周 · 总 193.0k star*
  - 📌 **是什么**：以 agent skills/rules 形式约束 AI 编码助手少写代码、按 YAGNI 做取舍。
  - 💡 **学习点**：学习如何通过规则文件控制 agent 的生成边界和决策质量。
  - 🧭 **上手**：对比同一任务在启用规则前后的生成 diff，重点分析省略了哪些代码。
- [affaan-m/ECC（Agent 框架）](https://github.com/affaan-m/ECC) — *GitHub · JavaScript · +23.0k/周 · 总 305.5k star*
  - 📌 **是什么**：为多种 AI coding agent 提供 harness 优化体系，覆盖技能、记忆、安全与研究优先流程。
  - 💡 **学习点**：学习如何把 agent 从“会聊天”提升为可控、可复盘、可观测的开发执行体。
  - 🧭 **上手**：查看其 skills 与 memory 配置样例，在本地 agent 中先加最小约束。
- [obra/superpowers（Agent Skills）](https://github.com/obra/superpowers) — *GitHub · Shell · +18.1k/周 · 总 319.3k star*
  - 📌 **是什么**：一套 agentic skills 框架与软件开发方法论，强调可落地的 agent 工作流。
  - 💡 **学习点**：学习把需求澄清、设计、实现、评审拆成 agent 可执行技能。
  - 🧭 **上手**：从一个 README 示例入手，用一个小需求走完 brainstorming 到实现流程。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向同步递归防护
- **技能点**：掌握 Vue 状态与派生数据双向同步时的循环控制能力，能定位递归更新根因并设计等值守卫。
- **坑点**：状态变化写入新数组引用，deep watch 又回写原状态，即使值未变也会互相唤醒，最终触发 Maximum recursive updates。
- **解决方案**：每次同步前先比较新旧结果，等价就不写入；回写状态时也只处理真正发生变化的字段。
```text
const same = (a, b) => a.length === b.length && a.every((v, i) => v === b[i])
watch(flags, () => {
  const next = buildSizes()
  if (same(next, localPanelSizes.value)) return
  localPanelSizes.value = next
})
watch(localPanelSizes, (sizes) => {
  if (isFileFolded.value !== isHidden(sizes[0])) {
    isFileFolded.value = isHidden(sizes[0])
  }
}, { deep: true })
```
- **拓展**：可沉淀为双向同步模板：统一使用 pure derive + equality guard，复杂场景优先改单向依赖。
- *来源：admin-workspace-new | 2026-08-11*

### 2. 组件事件冒泡边界控制
- **技能点**：巩固事件传播边界意识，能判断问题应修在共享组件还是业务组件，避免为局部行为污染通用能力。
- **坑点**：子组件点击事件冒泡到外层容器点击处理器，导致复制操作误触发解析展开或收起。
- **解决方案**：在业务调用处使用 @click.stop 阻断冒泡，共享复制组件保持无副作用的通用语义。
```text
<CopyText
  :text="'题目编号: ' + data.questionId"
  :copyText="data.questionId"
  @click.stop
/>
```
- **拓展**：后续可为可交互嵌套组件制定事件契约：明确哪些动作必须阻止冒泡，哪些必须透传。
- *来源：admin-workspace-new | 2026-08-11*

### 3. 工具栏配置类型化收敛
- **技能点**：掌握用常量和联合类型收敛动态配置的能力，降低组件接入方的认知成本。
- **坑点**：默认按钮与自定义按钮边界不清，业务只靠字符串拼装 toolbar-order，且公式等能力依赖 customButtons 才能注册。
- **解决方案**：补全默认按钮映射，将内置 handler 无条件注册，并导出按钮常量与联合类型供业务直接选用。
```text
export const TOOLBAR_BUTTONS = {
  formula: 'formula',
  fillInInsert: 'fill-in'
} as const

export type ToolbarButtonName = keyof typeof TOOLBAR_BUTTONS

interface Props {
  toolbarOrder?: (ToolbarButtonName | string)[]
}
```
- **拓展**：可延伸为组件配置注册中心：每个可用能力都有类型、默认 handler 和样式来源。
- *来源：admin-workspace-new | 2026-08-11*

### 4. 通用组件状态分层精简
- **技能点**：巩固组件职责分层能力，能区分通用组件能力和业务页面临时状态，避免过度同步。
- **坑点**：为折叠和显隐同时维护业务标志、持久化 key、基础宽度和双向 watch，导致逻辑链路长且易循环。
- **解决方案**：只保留拖拽面板的内部 sizes 状态和最小宽度，显隐由业务条件 v-if 控制，去掉无必要的持久化与双向同步。
```text
const localPanelSizes = ref([760, 'auto', 400])

<ResizablePanels
  v-model:sizes="localPanelSizes"
  :collapsible="[true, false, true]"
/>

<template v-if="isAiReviewer" #panel-2>
  <AiPanel />
</template>
```
- **拓展**：可形成分栏组件接入规范：展示显隐走业务，尺寸交互走组件内部状态。
- *来源：admin-workspace-new | 2026-08-11*

### 5. 多副本部署一致性排查
- **技能点**：掌握跨项目副本和部署环境下的事实校验习惯，先确认目标代码是否存在、线上分支是否一致再诊断。
- **坑点**：多份本地 clone 可能停在不同分支，线上问题也可能来自未提交代码，直接按本地记忆修改会排查方向错误。
- **解决方案**：引用前确认目标副本文件存在，排查线上现象时先用 git show 对比部署分支，避免本地与线上口径不一致。
```text
git show master:src/service/upload.ts | grep "getStsToken"
fs.existsSync("src/components/quillEditorNew")
```
- **拓展**：可沉淀为排障前置清单：副本、分支、部署 commit、关键文件、复现路径。
- *来源：admin-workspace-hr | 2026-09-24*

### 6. 空状态责任就近划分
- **技能点**：掌握 UI 空状态归属判断能力，能避免父子组件重复兜底导致逻辑冗余和样式不一致。
- **坑点**：父组件对子组件已有空状态的功能再次兜底，造成重复判断；而列表字段空态又缺少统一默认展示。
- **解决方案**：父组件只负责列表字段或数据块是否需要渲染，子组件负责自身完整空态，保持空状态逻辑就近内聚。
```text
<div v-if="q.missingInfoList?.length">
  <MissingInfo :list="q.missingInfoList" />
</div>
<div v-else>暂无缺失</div>

<AiGhostWordCard :data="q.ghostWords" />
```
- **拓展**：可提炼空状态模板规范：按字段空态、卡片空态、区域空态三层分别设计。
- *来源：admin-workspace-new | 2026-08-11*

