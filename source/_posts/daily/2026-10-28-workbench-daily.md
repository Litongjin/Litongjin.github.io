---
title: "工作台日报 · 2026-10-28"
date: 2026-10-28 07:03:24
categories: [工作日记]
tags: ["日报", "AI Agent", "大模型", "开发工具", "软件工程"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-10-28

## 🔥 行业热点

- [Wikipedia's Cleanup of AI-Generated Content](https://spectrum.ieee.org/wikipedia-wikiproject-ai-writing-cleanup) — *Hacker News*
  - 📌 **内容**：维基百科正在清理条目中由 AI 生成的低质量内容，反映出开放协作社区面对大模型批量产出时遇到的治理难题。
  - 💡 **学习**：可以了解 AI 生成文本的识别方式与社区审核流程，并思考如何为产品设计内容溯源与质量标注机制。
  - 🧭 **拓展**：可以对比维基百科的来源引用规范与自家内容平台的审核策略，尝试搭建一套简易的 AI 内容标记流程。
- [The Mythical Man-Month in the Age of AI](https://j11g.com/the-mythical-man-month-in-the-age-of-ai) — *Hacker News*
  - 📌 **内容**：文章借用《人月神话》的经典命题，探讨 AI 时代软件团队的规模、沟通成本与效率之间的关系。
  - 💡 **学习**：可以理解 AI 辅助编码如何改变分工与协作成本，并反思“加人或者加模型就能加速”这一直觉。
  - 🧭 **拓展**：可以在小团队中记录引入 AI 编码工具前后的交付节奏，做一次轻量复盘。
- [Anthropic trained Claude on illegally obtained Stanford professors' books](https://stanforddaily.com/2026/10/27/professors-books-pirated-to-train-claude/) — *Hacker News*
  - 📌 **内容**：该话题围绕大模型训练数据来源的合法性争议，涉及未经授权获取书籍并用于训练的问题。
  - 💡 **学习**：可以关注训练数据的版权与合规风险，以及数据供应链审计的必要性。
  - 🧭 **拓展**：可以为自己的数据集建立来源台账与许可证清单，并验证其是否可追溯。
- [Is my data used for model training?](https://privacy.claude.com/en/articles/7996868-is-my-data-used-for-model-training) — *Hacker News*
  - 📌 **内容**：文章讨论用户数据是否会被用于模型训练，以及相应的透明度与控制选项。
  - 💡 **学习**：可以学习产品中数据用途告知与选择退出机制的设计思路。
  - 🧭 **拓展**：可以检查自己所用服务的隐私设置与数据使用条款，验证其中是否提供退出通道。
- [Tailscale SSH: Investigation and remediation update](https://tailscale.com/ssh-investigation-update) — *Hacker News*
  - 📌 **内容**：该条针对 Tailscale SSH 安全事件，说明原因排查与修复进展。
  - 💡 **学习**：可以学习安全事件响应中的根因分析、修复与对外沟通方式。
  - 🧭 **拓展**：可以对照自身系统的 SSH 访问控制与审计日志，检查是否存在类似风险点。
- [Here be monsters: What do you still need to know when agents do the work?](https://blog.adrq.dev/here-be-monsters-what-do-you-still-need-to-know-when-agents-do-the-work) — *Hacker News*
  - 📌 **内容**：文章探讨当 AI Agent 承担实际工作后，人类仍然需要掌握哪些知识与把关能力。
  - 💡 **学习**：可以思考 Agent 工作流中的可观测性、验证与责任边界如何设计。
  - 🧭 **拓展**：可以为 Agent 任务增加人工审核节点与结果校验，并进行小规模对照实验。
- [Survival LLM: AI assistant and emergency library that runs on your own computer](https://www.brangus.io/survival-llm) — *Hacker News*
  - 📌 **内容**：该项目是一个可以完全在本地运行的 LLM 助手与应急资料库，强调离线可用与自持。
  - 💡 **学习**：可以了解本地模型部署、离线知识库以及资源受限环境下推理时的取舍。
  - 🧭 **拓展**：可以在本机用小型开源模型搭建离线问答，验证断网场景下的可用性。
- [Show HN: Specification First Agentic Development Plugin and Prove](https://spec-first.dev/) — *Hacker News*
  - 📌 **内容**：该插件主张“规格先行”，尝试让 AI 编码从需求规格出发，并产出可验证的结果。
  - 💡 **学习**：可以学习以规格或契约驱动 Agent 编码的工作流，以及如何设计验证环节。
  - 🧭 **拓展**：可以在小项目中用规格文档驱动 Agent 生成代码，再用测试用例验证。
- [One year of Dioxus fullstack in production: the good, the painful](https://oxidt.com/blog/one-year-of-dioxus-fullstack-in-production-the-good-the-painful) — *Hacker News*
  - 📌 **内容**：该条分享在生产环境中使用 Rust 全栈框架 Dioxus 一年的经验与遇到的问题。
  - 💡 **学习**：可以了解 Rust 前端和全栈方案的成熟度、构建体验与性能方面的权衡。
  - 🧭 **拓展**：可以用 Dioxus 做一个全栈小应用，实测编译时间与开发体验。
- [Ghostty 1.4](https://ghostty.org/docs/install/release-notes/1-4-0) — *Hacker News*
  - 📌 **内容**：终端模拟器 Ghostty 发布新版本，通常涉及性能、兼容性与配置体验方面的改进。
  - 💡 **学习**：可以了解现代终端在渲染性能、跨平台与配置方式上的设计取向。
  - 🧭 **拓展**：可以本地安装试用，并与现有终端对比启动速度与渲染表现。

## 🌟 GitHub 热门开源项目

- [morluto/rea（Agent Skills）](https://github.com/morluto/rea) — *GitHub · TypeScript · +341.7k/周 · 总 397.1k star*
  - 📌 **是什么**：一套让 AI Agent 参与逆向工程的技能与工具集，覆盖从应用行为观察到原生二进制、反编译分析的链路。
  - 💡 **学习点**：学习如何把 Ghidra 这类重型领域工具封装成 Agent 可编排的 CLI 步骤。
  - 🧭 **上手**：先读仓库里对接反汇编/反编译工具的那层适配代码，再拿一个简单二进制文件跑通最小分析流程。
- [Panniantong/Agent-Reach（Agent Skills）](https://github.com/Panniantong/Agent-Reach) — *GitHub · Python · +19.3k/周 · 总 104.9k star*
  - 📌 **是什么**：一个 Python 命令行工具，让 Agent 能读取和搜索 Twitter、Reddit、YouTube、GitHub、B 站、小红书等平台内容，号称不依赖付费 API。
  - 💡 **学习点**：学习如何把多个平台的数据抓取统一成 Agent 友好的单一 CLI 接口与输出格式。
  - 🧭 **上手**：先跑通 README 里最简单的单平台搜索命令，再看它如何把结果整理成 Agent 易读的结构化输出。
- [Leonxlnx/taste-skill（Agent Skills）](https://github.com/Leonxlnx/taste-skill) — *GitHub · JavaScript · +13.3k/周 · 总 103.6k star*
  - 📌 **是什么**：一组给编码 Agent 补“审美”的技能，通过艺术指导、参考图和浏览器校验，避免生成千篇一律的 AI 风格页面。
  - 💡 **学习点**：前端工程师可学习如何把设计规范与视觉验收标准写成 Agent 可执行的技能和自检回路。
  - 🧭 **上手**：用同一句提示分别在有/无该技能下生成一个落地页，对比它的浏览器校验环节抓出了哪些差异。
- [farion1231/cc-switch（开发工具）](https://github.com/farion1231/cc-switch) — *GitHub · Rust · +12.2k/周 · 总 149.4k star*
  - 📌 **是什么**：一个 Rust 写的跨平台桌面工具，集中管理 Claude Code、Codex、OpenCode 等多个编码 Agent 的配置与切换。
  - 💡 **学习点**：学习多款 Agent CLI 的配置结构与切换方式，为自己的工具链做统一管理。
  - 🧭 **上手**：装好后把本机已有的两三个编码 Agent 配置导入，观察它如何抽象各家不同的配置格式。
- [addyosmani/agent-skills（Agent Skills）](https://github.com/addyosmani/agent-skills) — *GitHub · JavaScript · +11.8k/周 · 总 111.1k star*
  - 📌 **是什么**：一套面向 AI 编码 Agent 的“生产级工程技能”集合，把成熟工程实践沉淀为可调用的技能。
  - 💡 **学习点**：学习资深工程师如何把评审、测试、性能等经验拆解成 Agent 可复用的指令与流程。
  - 🧭 **上手**：挑一个最贴近你当前工作的技能文件（如代码评审），读它的结构后接进 Claude Code 或 Cursor 实跑一次。

## 🚀 技能提升点（工作总结汇总）

### 1. watch 双向同步的循环守卫
- **技能点**：掌握 Vue 中「状态A→数组→状态B→状态A」双向 watch 的失效链路分析与修复，能在每一步写回前加值比较，从而彻底消除循环。
- **坑点**：两个 watch 互相回写：flag 变化写入新数组引用，deep watch 再写回 flag；即使最终值没变，也会因引用变化和深度监听反复唤醒，直到报错 Maximum recursive updates exceeded。
- **解决方案**：在两端各处加守卫：计算出 next 后先比较内容，相等时直接 return；回写 flag 前先比较当前值与推导值，不等才赋值。
```text
const next = [leftWidth, 'auto', rightWidth]
if (next[0] === localPanelSizes.value[0] && next[2] === localPanelSizes.value[2]) return
localPanelSizes.value = next
// watch(localPanelSizes) 内
if (isFilePreviewFolded.value !== derivedFolded) isFilePreviewFolded.value = derivedFolded
```
- **拓展**：复杂同步链路优先改为单向数据流（状态派生视图），或改用 computed 加显式事件，从根本上避免双向 watch。
- *来源：admin-workspace-new / batchInput.vue，2026-08-11*

### 2. 共享组件的冒泡边界
- **技能点**：定位点击事件冒泡导致的连带交互：能判断是容器监听误捕获，还是组件自身行为有问题。
- **坑点**：共享组件 CopyText 的点击冒泡到外层容器的 @click（展开/收起题目），导致「复制」被误触发为「展开」。若在共享组件里加 stopPropagation，会污染所有调用方。
- **解决方案**：保持共享组件通用：不改内部行为，在业务组件调用处加修饰符 @click.stop，并用 prop 分离「显示文本」与「实际复制的值」。
```text
<CopyText :text="`题目编号: ${data.questionId}`" :copyText="data.questionId" @click.stop />
```
- **拓展**：沉淀约定：通用组件不内置 stopPropagation；消费方一律显式 @click.stop，或在容器上用 @click.self。
- *来源：admin-workspace-new / questionCard.vue，2026-08-11*

### 3. 插槽透传扩展共享组件
- **技能点**：在不侵入第三方或共享组件默认逻辑的前提下，用插槽透传（配合 $slots 探测）为调用方提供自定义渲染点。
- **坑点**：直接改共享组件来定制折叠图标，会影响所有使用方；而官方折叠按钮又无法从外部覆盖。
- **解决方案**：组件内新增具名插槽透传，用 $slots 判断是否存在该插槽，只有存在时才隐藏原生图标；业务侧通过插槽注入 svgIcon，并用 CSS rotate 区分展开和收起状态。
```text
<template v-if="$slots['start-collapsible']" #start-collapsible>
  <slot name="start-collapsible" />
</template>
.collapse-icon.is-folded { transform: rotate(180deg) }
```
- **拓展**：把「图标、文案、空状态」统一收敛为插槽约定，形成组件对外扩展的固定约定。
- *来源：admin-workspace / ResizablePanels，2026-08-10*

### 4. 组件能力按需裁剪
- **技能点**：按最小需求裁掉不必要状态：识别「记忆/持久化/双向同步」是否真需要，删掉状态与联动，只留纯局部状态。
- **坑点**：在拖拽面板上叠加尺寸持久化、折叠标志与双向同步，导致状态源多头、watch 链路复杂且易死循环；用 sizes 里的 "hidden" 字符串控制显隐也不直观。
- **解决方案**：删除 storageKey 传参、折叠标志 ref、BASE 常量与双向 watch，localPanelSizes 只作组件内部状态；右栏显隐改由 isAiReviewer + v-if 插槽控制，通用组件的 storageKey 能力保留（空 key 时不写入）。
```text
// 调用方不再传 storageKey
<ResizablePanels :sizes="localPanelSizes" :panelCollapsible="[true,false,true]" />
<template v-if="isAiReviewer" #panel-2>...</template>
```
- **拓展**：通用组件保留全部能力、调用方按需传参；删除后 grep 确认无符号残留引用。
- *来源：admin-workspace-new / batchInput.vue，2026-08-11*

### 5. 工具栏按钮的常量化注册
- **技能点**：把散落在多个文件里的字符串标识统一为常量与联合类型导出，并让默认能力无需额外配置即可用。
- **坑点**：业务方不知道 toolbar-order 支持哪些名字；formula、fillInInsert 等必须先配 customButtons 才生效，string 类型让拼写错误无法在编译期发现。
- **解决方案**：补全默认按钮映射，把 handler 改为无条件注册；新增 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 类型，调用方用 B.xxx 常量声明，类型收敛为 ToolbarButtonName | string。
```text
export const TOOLBAR_BUTTONS = ['undo','redo','divider','fillInInsert','formula','image'] as const
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[number]
// type.ts
toolbarOrder?: ToolbarButtonName | string
```
- **拓展**：再往前一步可做按名字的运行时校验与默认顺序，IDE 里即可补全所有可用按钮。
- *来源：admin-workspace-new / QuillEditorNew，2026-08-11*

### 6. 线上现象与部署分支核对
- **技能点**：建立「先确认跑的是哪份代码」的排查习惯，避免在本地最新代码里空找不存在的问题。
- **坑点**：排查线上异常时直接看工作区代码，但测试环境部署的是 master 旧码，新改动全在未提交工作区，现象与代码对不上。
- **解决方案**：先用 git show master:<文件> 或 git log 对比部署分支版本，再判断是环境问题还是代码缺陷。
```text
git log --oneline -1 master
git show master:src/pages/upload/index.tsx | head -50
```
- **拓展**：配合在关键页面暴露版本号/build 标识，或记录发布 commit，进一步缩短定位时间。
- *来源：admin-workspace-hr，2026-09-24*

### 7. 编辑落盘与会话切换校验
- **技能点**：掌握 AI 协作编辑的可靠性校验：会话切换、并行编辑、上下文丢失都可能造成改动未写入磁盘，必须显式复核。
- **坑点**：会话切换后，前期对 toolbarConfig.ts、detailComment.vue 的 edit 实际未写入磁盘（磁盘仍是旧版），却在总结里被当作已完成；同一文件并行编辑基于同一快照会互相覆盖。
- **解决方案**：完成一批改动后用 grep/读文件核对关键符号是否真实存在，缺失则整体重做；同一文件的多个编辑严格逐条顺序执行。
```text
grep -rn "syncSizesFromFlags\|LEFT_BASE\|isAiCheckContentFolded" src/  # 期望无输出
grep -n "TOOLBAR_BUTTONS" src/components/quillEditorNew/toolbar/*.ts  # 期望命中
```
- **拓展**：收尾固定跑一次「已删符号 0 残留 + 新增符号全部命中」的自检清单，作为提交前必跑步骤。
- *来源：admin-workspace-new，2026-08-11*

