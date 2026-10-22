---
title: "工作台日报 · 2026-10-23"
date: 2026-10-23 07:02:06
categories: [工作日记]
tags: ["日报", "AI", "AI Agent", "大模型", "安全"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-10-23

## 🔥 行业热点

- [OpenShell – A safe, private runtime for fleets of autonomous AI agents](https://github.com/NVIDIA/OpenShell) — *Hacker News*
  - 📌 **内容**：一个面向大规模自治 AI Agent 的安全私有运行时，主打隔离与隐私保护，适合企业级 Agent 集群部署。
  - 💡 **学习**：了解 Agent 运行时的沙箱隔离、权限管控设计思路，为自己搭建多 Agent 系统提供安全架构参考。
  - 🧭 **拓展**：可对比 E2B、Firecracker 等沙箱方案，动手实验其隔离边界与资源开销。
- [Show HN: CATT 0.2.1: Test agents/services against a Muse/Instinct-alike, in CI](https://github.com/Coverage-Cat/consumer-agent-testing-tool) — *Hacker News*
  - 📌 **内容**：CATT 是一个在 CI 中针对类 Muse/Instinct 模型或服务进行自动化测试的工具，帮助把 AI 行为纳入持续集成。
  - 💡 **学习**：学习如何为 AI Agent/模型服务编写可回归的 CI 测试，把'AI 行为验证'工程化。
  - 🧭 **拓展**：可研究其断言与回放机制，尝试接入自己的 Agent 流水线做回归测试。
- [Jailbreaking the Switch 2](https://pepsipu.com/blog/2026-10-switch2-jailbreak-ns2jb/) — *Hacker News*
  - 📌 **内容**：Switch 2 被越狱的技术文章，涉及新一代主机硬件的安全机制与漏洞利用过程。
  - 💡 **学习**：了解现代游戏主机的安全启动、固件签名与漏洞挖掘思路，对逆向和硬件安全有兴趣者可深入。
  - 🧭 **拓展**：可延伸阅读可信启动链（Chain of Trust）与 glitching 攻击的相关资料。
- [WorldTUI: I trained a 360M model to pretend to be a terminal](https://simedw.com/2026/10/21/world-model-terminal/) — *Hacker News*
  - 📌 **内容**：作者训练了一个 360M 参数的小模型来模拟终端行为，探索用小模型'扮演'交互式环境的世界模型思路。
  - 💡 **学习**：学习小参数模型的专项训练方法，以及'模型即环境模拟器'这一 world model 实验方向。
  - 🧭 **拓展**：可复现该项目，尝试用类似方法模拟 REPL、游戏等交互环境并评估一致性。
- [Every AI model has the same "controversial" take on humans](https://www.modelcompare.ai/blog/ai-models-controversial-take-on-humans) — *Hacker News*
  - 📌 **内容**：文章观察到一个现象：各家的 AI 模型在涉及人类议题时会给出趋同的'争议性'回答，反映训练数据与对齐的共同倾向。
  - 💡 **学习**：帮助理解模型对齐与训练分布如何导致输出趋同，提示在做评测时注意跨模型偏差。
  - 🧭 **拓展**：可自己设计同一组敏感提示，横向测试多家模型验证该现象。
- [AI and the Enclosure of the Scientific Commons](https://arxiv.org/abs/2610.27251) — *Hacker News*
  - 📌 **内容**：讨论 AI 时代科学知识公共领域被'圈占'的问题，涉及论文数据、模型训练与学术开放之间的张力。
  - 💡 **学习**：思考训练数据版权与开放科学的关系，对构建依赖学术数据的 AI 产品有合规参考价值。
- [AI analysis of CT scans finds 42.9% of patients have coronary inflammation](https://www.biograph.com/newsroom/biograph-completes-first-u.s.-study-with-caristo-diagnostics%E2%80%99-ai-powered-coronary-inflammation-detection-finding-risk-in-over-42-of-asymptomatic-patients) — *Hacker News*
  - 📌 **内容**：利用 AI 分析 CT 影像发现相当比例患者存在冠状动脉炎症，展示 AI 在医学影像辅助诊断中的进展。
  - 💡 **学习**：了解医学影像 AI 的应用形态与技术路线（影像分割/风险量化），对跨界做医疗 AI 的工程师有参考。
- [Show HN: UI micro interactions prompts for AI](https://www.typeui.sh/micro-interactions) — *Hacker News*
  - 📌 **内容**：一套用于 AI 的 UI 微交互动效提示词集合，帮助用 AI 生成高质量的界面微交互代码。
  - 💡 **学习**：学习如何用结构化提示词驱动 AI 生成前端动效（如悬停、过渡、加载反馈），提升 AI 辅助前端开发质量。
  - 🧭 **拓展**：可将这些提示词接入自己的 AI 编码工作流（如 Cursor），对比生成动效质量。
- [Award-winning AI literature](https://jamesyu.org/award-winning-ai-literature/) — *Hacker News*
  - 📌 **内容**：关于获奖的 AI 生成文学作品的讨论，涉及 AI 写作能力边界与文学评价标准的争议。
  - 💡 **学习**：可借此思考大模型长文写作的能力现状与评估方式，对做内容生成类产品有启发。
- [Mini brain organoids ‘reading’ Braille key step towards biohybrid computing](https://www.frontiersin.org/news/2026/10/16/mini-brain-organoids-reading-braille-biohybrid-computing-frontiers-artificial-intelligence) — *Hacker News*
  - 📌 **内容**：研究人员让微型脑类器官'读取'盲文信号，被视为迈向生物混合计算（湿件计算）的关键一步。
  - 💡 **学习**：了解生物混合计算这一前沿方向：用活体神经组织做信息处理，拓展对'计算'形态的认知。
  - 🧭 **拓展**：可延伸阅读 organoid intelligence（类器官智能）相关论文与伦理讨论。

## 🌟 GitHub 热门开源项目

- [vectorize-io/hindsight（Agent 框架）](https://github.com/vectorize-io/hindsight) — *GitHub · Python · +11.8k/周 · 总 52.7k star*
  - 📌 **是什么**：一个面向智能体的记忆系统，主打让 Agent 能够从交互中学习并持续积累记忆。
  - 💡 **学习点**：学习 Agent 长期记忆的设计模式，理解记忆如何存储、检索和反馈到推理循环中。
  - 🧭 **上手**：阅读核心记忆模块源码，对比它与简单向量检索方案的差异。
- [K-Dense-AI/scientific-agent-skills（Agent Skills）](https://github.com/K-Dense-AI/scientific-agent-skills) — *GitHub · Python · +7.9k/周 · 总 53.9k star*
  - 📌 **是什么**：面向科学研究的 Agent Skills 库，提供大量即用的技能与科学数据库接入，覆盖生物信息、化学信息等领域。
  - 💡 **学习点**：学习如何为特定领域编写结构化、可验证的 Agent Skill，理解 Skill 的输入输出规范。
  - 🧭 **上手**：挑选一个生物信息学 Skill，阅读其定义文件并尝试在 Claude Code 中调用。
- [usestrix/strix（Agent 框架）](https://github.com/usestrix/strix) — *GitHub · Python · +7.1k/周 · 总 71.2k star*
  - 📌 **是什么**：开源 AI 渗透测试工具，利用 Agent 自动发现并修复应用漏洞，面向安全测试场景。
  - 💡 **学习点**：学习安全领域 Agent 的工具编排与多步推理设计，了解如何将 LLM 接入渗透测试流程。
  - 🧭 **上手**：跑一遍官方提供的靶场示例，观察 Agent 如何分阶段执行漏洞探测。
- [Shubhamsaboo/awesome-llm-apps（Agent 框架）](https://github.com/Shubhamsaboo/awesome-llm-apps) — *GitHub · Python · +3.4k/周 · 总 142.7k star*
  - 📌 **是什么**：汇集 100+ 开源 AI Agent、Agent Skills 与 RAG 应用的精选合集，覆盖多种场景与框架。
  - 💡 **学习点**：快速浏览各类 LLM 应用架构，建立对 Agent 与 RAG 常见模式的整体认知。
  - 🧭 **上手**：按目录挑选一个与前端场景相关的项目，阅读其 README 并本地运行。
- [coreyhaines31/marketingskills（Agent Skills）](https://github.com/coreyhaines31/marketingskills) — *GitHub · JavaScript · +2.4k/周 · 总 56.0k star*
  - 📌 **是什么**：为 Claude Code 及 AI Agent 提供营销技能的插件集，涵盖转化率优化、文案、SEO 与增长工程。
  - 💡 **学习点**：学习如何将非技术领域知识（如营销）结构化为可复用的 Agent Skill。
  - 🧭 **上手**：阅读某个 SEO Skill 的定义文件，理解其提示词与工具调用设计。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向 watch 死循环
- **技能点**：掌握了 Vue 响应式系统中双向状态同步的调试与修复方法，理解了 deep watch 对引用类型（数组/对象）的敏感性。
- **坑点**：状态 A 派生数组 → watch 数组（deep）回写状态 B → 状态 B 又触发 A 的 watch。即使标志值最终未变，每次生成的新数组引用也会触发 deep watch，导致无限递归报 'Maximum recursive updates exceeded'。
- **解决方案**：在每一步同步前做值比较守卫：计算 next 后先逐项比对，完全相等则直接 return 不写回；回写标志前也比较当前值与推导值，无变化不赋值。
```text
function syncSizesFromFlags() {
  const next = [leftWidth, 'auto', rightWidth]
  const cur = localPanelSizes.value
  if (cur[0] === next[0] && cur[1] === next[1] && cur[2] === next[2]) return
  localPanelSizes.value = next
}

watch(localPanelSizes, (sizes) => {
  const folded = sizes[0] === 0
  if (isFilePreviewFolded.value !== folded) isFilePreviewFolded.value = folded
}, { deep: true })
```
- **拓展**：可沉淀为通用模式：任何「状态 ↔ 派生对象 ↔ 状态」的 watch 链都必须加值比较守卫，或改用 computed 单向派生。
- *来源：admin-workspace 2026-08-11*

### 2. 共享组件保持通用，业务侧拦截冒泡
- **技能点**：掌握了组件库设计中的职责边界：共享组件保持纯净通用，特殊交互需求在业务调用方处理。
- **坑点**：CopyText 组件点击事件冒泡到外层容器触发展开/收起，最初的修复方向是改共享组件加 stopPropagation，但这会让所有调用方行为被强制改变。
- **解决方案**：在业务组件的调用处加 @click.stop 拦截冒泡，共享组件 copyText.vue 回退保持通用（handleCopy 无 event 参数、无 stopPropagation）。
```text
<CopyText :text="`题目编号: ${data.questionId}`" :copyText="data.questionId" @click.stop />
```
- **拓展**：共享组件只做一件事并做好，业务差异在调用方收口——这是组件库设计的通用原则。
- *来源：admin-workspace 2026-08-11*

### 3. 编辑器工具栏声明式注册
- **技能点**：掌握了富文本编辑器工具栏的可配置化改造：从「必须传 customButtons 才能用」到「仅通过 toolbar-order 声明即可启用」。
- **坑点**：业务组件不知道 :toolbar-order 能传哪些按钮名，公式/填空等按钮必须额外配置 customButtons 才能注册 handler，使用门槛高。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 联合类型，toolbar-order 类型改为 ToolbarButtonName | string；内置 handler 改为无条件注册，不再依赖 customButtons。
```text
export const TOOLBAR_BUTTONS = {
  undo: 'undo', redo: 'redo', divider: 'divider',
  formula: 'formula', fillInInsert: 'fillInInsert',
  bold: 'bold', italic: 'italic', image: 'image'
} as const
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[keyof typeof TOOLBAR_BUTTONS]
```
- **拓展**：可推广到所有可配置组件：导出常量 + 联合类型让调用方获得 IDE 自动补全与类型检查，降低接入成本。
- *来源：admin-workspace-new 2026-08-11*

### 4. 删繁就简：去掉过度设计
- **技能点**：掌握了识别并移除过度设计的能力：当需求从「记忆+持久化+双向同步」简化为「仅拖拽+折叠」时，果断删除所有相关状态与逻辑。
- **坑点**：保留了 storageKey 持久化、双向 watch 同步、折叠标志 ref 等一整套复杂机制，但实际需求只需要基础拖拽和折叠功能。
- **解决方案**：删除 storageKey 传参、折叠标志 ref、syncSizesFromFlags 及全部双向 watch；localPanelSizes 降级为纯内部状态，右栏显隐改用 v-if 控制。验证删除符号 0 处残留引用。
```text
// 删除前：storageKey + 双向 watch + 折叠标志
// 删除后：仅保留纯拖拽内部状态
const localPanelSizes = ref([760, 'auto', 400])
const panelCollapsible = [true, false, true]
```
- **拓展**：定期审视代码是否承载了超出当前需求的设计，及时删减比保留「以防万一」更有价值。
- *来源：admin-workspace 2026-08-11*

### 5. 组件插槽透传自定义折叠图标
- **技能点**：掌握了在封装组件中通过插槽透传 + 条件检测来支持业务方自定义 UI 细节的模式。
- **坑点**：需要自定义折叠图标样式，但折叠行为由底层 el-splitter-panel 原生处理，不能直接在业务层覆盖。
- **解决方案**：在 ResizablePanels 组件新增 #start-collapsible 插槽透传（用 $slots 条件检测），隐藏官方折叠图标；业务组件通过 #panel-{i}-start-collapsible 插槽注入自定义图标 + CSS rotate 区分收起/展开态。
```text
<template #panel-0-start-collapsible>
  <svgIcon :name="collapsed ? 'arrow-right' : 'arrow-left'" class="collapse-icon" />
</template>
```
- **拓展**：封装组件的定制点设计：优先用插槽而非 props 传 UI 片段，保持灵活性。
- *来源：admin-workspace 2026-08-10*

### 6. CSS 卡片分隔方案迭代
- **技能点**：掌握了卡片列表视觉分隔的多种 CSS 方案及其取舍，理解了从「整体边框 → 相邻分隔线 → 上下边框+间距」的迭代思路。
- **坑点**：初版方案（整体边框或相邻分隔线 & + &）在实际布局中效果不佳，需要多轮调整才能满足视觉要求。
- **解决方案**：最终采用四周 border + border-radius + margin 间距的方案：每卡片独立边框圆角，卡片间用 margin 留白，删掉冗余的 first-child/last-child 特殊处理。
```text
.question-item {
  border: 1px solid #ebecf2;
  border-radius: 4px;
  margin: 12px 8px;
  &:last-child { margin-bottom: 0 }
}
```
- **拓展**：卡片分隔首选「独立边框 + margin 间距」方案，避免相邻选择器带来的边界 case。
- *来源：admin-workspace 2026-08-11*

