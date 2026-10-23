---
title: "工作台日报 · 2026-10-24"
date: 2026-10-24 07:05:23
categories: [工作日记]
tags: ["日报", "大模型", "AI工具", "互联网产品", "安全"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-10-24

## 🔥 行业热点

- [WorldTUI: I trained a 360M model to pretend to be a terminal](https://simedw.com/2026/10/21/world-model-terminal/) — *Hacker News*
  - 📌 **内容**：作者训练了一个 360M 的小模型来模拟终端行为，让模型学习命令行交互的模式与输出，展示了小模型在特定领域的拟合能力。
  - 💡 **学习**：可以学习如何用小参数模型做领域特定的行为模拟，包括训练数据构造和序列建模的思路。
  - 🧭 **拓展**：可尝试用类似方法训练小模型模拟 REPL、SQL 终端等，验证小模型在窄域任务上的可行性。
- [Now Google says it might use your Gmail and Google Photos data to target ads](https://www.theverge.com/tech/1017600/google-personal-intelligence-privacy-policy-ads-update) — *Hacker News*
  - 📌 **内容**：谷歌更新了隐私相关政策表述，暗示可能利用 Gmail 和 Google Photos 的用户数据进行广告定向，引发对互联网大厂数据使用边界的讨论。
  - 💡 **学习**：了解主流互联网产品的数据收集与广告定向机制，以及隐私政策变化对开发者和用户的影响。
  - 🧭 **拓展**：可延伸关注各国隐私合规要求（如 GDPR）对广告技术实现的约束。
- [Securing our frontier model training environment](https://openai.com/index/securing-our-frontier-model-training-environment/) — *Hacker News*
  - 📌 **内容**：介绍如何为前沿大模型的训练环境构建安全防护体系，涉及训练基础设施的隔离、访问控制与防泄漏措施。
  - 💡 **学习**：学习 AI 基础设施安全的实践要点：网络隔离、凭证管理、防止模型权重被窃取等。
  - 🧭 **拓展**：可对照自家 GPU 集群或云训练环境检查安全策略是否完备。
- [We have traits at home](https://www.axelmagn.com/blog/202610-zig-traits/) — *Hacker News*
  - 📌 **内容**：文章借用流行的梗句式，讨论如何在编程语言中自行实现类似 Rust traits 的能力，面向没有原生 trait 系统的语言给出替代方案。
  - 💡 **学习**：了解 trait/类型类机制的本质，以及在 Go、C 或动态语言中用接口、组合或代码生成模拟它的技巧。
  - 🧭 **拓展**：可对比 Rust traits、Haskell typeclass、Go interface 的异同并动手实现一个简化版。
- [New AI Is More Than Twice as Good at Diagnosing Diseases as Doctors](https://gizmodo.com/microsoft-says-its-new-ai-is-more-than-twice-as-good-at-diagnosing-diseases-as-doctors-2000829562) — *Hacker News*
  - 📌 **内容**：新研究表明某 AI 系统在疾病诊断准确率上显著超过医生，体现了大模型/专科模型在医疗 AI 领域的进展。
  - 💡 **学习**：关注医疗诊断类 AI 的评测方式与模型设计思路，了解垂直领域模型如何与专业知识结合。
  - 🧭 **拓展**：可延伸了解医疗 AI 的基准测试集与落地中的合规、可解释性问题。
- [A whirlwind tour of systemd-nspawn containers](https://quantum5.ca/2025/03/22/whirlwind-tour-of-systemd-nspawn-containers/) — *Hacker News*
  - 📌 **内容**：快速介绍 systemd-nspawn 这一 systemd 自带的轻量容器工具，涵盖创建、启动与管理容器的核心用法。
  - 💡 **学习**：学习一种不依赖 Docker 的容器化方案，理解 chroot 升级版、命名空间隔离的基本原理。
  - 🧭 **拓展**：可在自己的 Linux 机器上用 nspawn 搭建隔离的构建或测试环境验证。
- [Show HN: Corgi, a helpful {agent, terminal} multiplexer](https://corgi.computer/) — *Hacker News*
  - 📌 **内容**：作者发布了一个名为 Corgi 的开源工具，用于同时复用/管理多个 AI agent 与终端会话，定位类似 tmux 但面向 agent 工作流。
  - 💡 **学习**：了解 AI 编码 agent 并行工作流的新工具形态，以及终端复用器与 agent 编排结合的设计思路。
  - 🧭 **拓展**：可试用该工具管理多个 Claude Code/Codex 会话，评估多 agent 并行开发的效率。
- [Local Wikipedia: Wikipedia, but only for things a half hour walk from you](https://edjefferson.com/localwiki/) — *Hacker News*
  - 📌 **内容**：一个项目将维基百科内容按地理位置过滤，只展示步行半小时范围内的条目，是一种有趣的本地化信息服务实验。
  - 💡 **学习**：学习基于地理坐标对大规模文本数据进行空间过滤与检索的实现方式，以及维基百科数据（Wikidata/GeoData）的使用。
  - 🧭 **拓展**：可用 Wikidata SPARQL 查询或离线维基数据复刻一个类似的地理半径检索应用。
- [Docker is weird](https://codeandcake.dev/posts/2026-10-22-docker-is-weird) — *Hacker News*
  - 📌 **内容**：文章吐槽并解析 Docker 在使用与实现层面的种种怪异之处，帮助读者理解容器工具背后反直觉的行为。
  - 💡 **学习**：深入理解 Docker 的分层文件系统、网络命名空间、守护进程模型等机制，明白坑从何而来。
  - 🧭 **拓展**：可对照 containerd、Podman 的实现差异，验证文中提到的怪异行为。
- [Today, we hacked what we'd set out to hack: the iPhone](https://blog.calif.io/p/iphone) — *Hacker News*
  - 📌 **内容**：安全研究团队宣布成功攻破 iPhone，达成了既定的破解目标，涉及 iOS 漏洞利用链的研究分享。
  - 💡 **学习**：了解移动端漏洞挖掘与利用链构造的思路，以及现代操作系统缓解措施（PAC、沙箱）对攻击的影响。
  - 🧭 **拓展**：可延伸学习公开的 iOS 漏洞 writeup 与 Pwn2Own 类赛事的利用案例。

## 🌟 GitHub 热门开源项目

- [tt-a1i/archify（Agent Skills）](https://github.com/tt-a1i/archify) — *GitHub · JavaScript · +18.4k/周 · 总 88.3k star*
  - 📌 **是什么**：一个将想法、计划或代码库自动转化为交互式架构图的 Agent Skill，可挂载到 Claude Code、Codex 等编码代理上使用。
  - 💡 **学习点**：学习如何把一个具体能力（图表生成）封装成 Agent 可调用的 Skill，理解 Skill 的提示词与上下文组织方式。
  - 🧭 **上手**：克隆后在 Claude Code 中加载该 skill，对一个自己的side project跑一遍生成架构图，观察它如何解析代码结构。
- [cathrynlavery/diagram-design（Agent Skills）](https://github.com/cathrynlavery/diagram-design) — *GitHub · HTML · +18.3k/周 · 总 60.3k star*
  - 📌 **是什么**：面向多种编码代理的编辑级图表设计技能，提供大量图表类型，输出自包含的 HTML+SVG 而非 Mermaid。
  - 💡 **学习点**：学习如何用纯前端产物（HTML+SVG）作为 Agent 的输出格式，体会『自包含、无依赖』交付物对 LLM 生成质量的约束设计。
  - 🧭 **上手**：对比它生成的 SVG 图表与你让模型直接写 Mermaid 的效果差异，理解为何作者选择自包含 HTML 路线。
- [rohitg00/ai-engineering-from-scratch](https://github.com/rohitg00/ai-engineering-from-scratch) — *GitHub · Python · +17.5k/周 · 总 72.8k star*
  - 📌 **是什么**：一套从零学习 AI 工程的课程体系，覆盖深度学习、生成式 AI、智能体等方向，强调边学边做。
  - 💡 **学习点**：适合作为转型路线图：按模块系统性补齐 LLM 与 Agent 的理论+实战，而非碎片式读博客。
  - 🧭 **上手**：先浏览仓库目录结构，从 agents 或 generative-ai 模块挑一个 lab 动手跑通，评估自己的知识缺口。
- [alibaba/open-code-review（开发工具）](https://github.com/alibaba/open-code-review) — *GitHub · Go · +8.4k/周 · 总 48.1k star*
  - 📌 **是什么**：阿里开源的混合架构代码评审工具：确定性流水线 + LLM Agent 结合，支持精确到行级的评审意见与多语言内置规则。
  - 💡 **学习点**：学习『规则流水线负责确定性检查、LLM 负责语义判断』的混合工程范式，这是企业级 AI 工具落地的典型架构。
  - 🧭 **上手**：阅读其 pipeline 与 agent 协作的核心模块，弄清行级评论是如何从 LLM 输出映射回 diff 的。
- [AstrBotDevs/AstrBot（Agent 框架）](https://github.com/AstrBotDevs/AstrBot) — *GitHub · Python · +2.1k/周 · 总 43.0k star*
  - 📌 **是什么**：多平台 IM 机器人与 Agent 开发框架，聚合多种 LLM、插件体系与 AI 能力，可作为自托管的智能助手底座。
  - 💡 **学习点**：学习如何设计『渠道接入层 + LLM 抽象层 + 插件系统』的聊天 Agent 架构，以及 MCP 工具在其中的接入方式。
  - 🧭 **上手**：用 Docker 跑起来接一个 Telegram/Discord 渠道，再写一个最小自定义插件体验其扩展机制。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步死循环
- **技能点**：掌握了 Vue 3 中 A→数组→B→A 双向 watch 同步的防御写法：每一跳都做值比较守卫，才能阻断互相唤醒。
- **坑点**：折叠标志变化 → 重建新数组引用赋值 localPanelSizes → deep watch 触发 → 回写标志，即使最终值未变，新数组引用也会让 watch 互踢，直至 'Maximum recursive updates exceeded'。
- **解决方案**：syncSizesFromFlags 计算 next 后与当前逐项比对，全等直接 return 不赋值；localPanelSizes 的 watch 内每个标志先比较 !== 再决定是否回写。
```text
function syncSizesFromFlags() {
  const next = [leftWidth, 'auto', rightWidth]
  const cur = localPanelSizes.value
  if (next.every((v, i) => v === cur[i])) return // 值没变就不写，断环
  localPanelSizes.value = next
}
watch(localPanelSizes, (sizes) => {
  const folded = sizes[0] === 0
  if (isFilePreviewFolded.value !== folded) isFilePreviewFolded.value = folded
}, { deep: true })
```
- **拓展**：任何「派生状态回写源状态」的链路都要先问：这一步在值没变时还会不会产生新引用/新写入。
- *来源：admin-workspace-new 2026-08-11*

### 2. 共享组件保持通用，业务侧拦截事件
- **技能点**：巩固了「共享组件不加业务特例」的边界意识：事件冒泡问题在用它的业务组件上 .stop，而不是给通用组件塞 stopPropagation。
- **坑点**：卡片内 CopyText 点击冒泡到外层容器 @click，误触发解析展开/收起；最初想改共享 copyText 组件，会污染所有调用方。
- **解决方案**：共享组件回退保持纯净，业务组件用 @click.stop 就地阻断冒泡；顺带用 copyText prop 区分「显示文本」与「实际复制内容」。
```text
<CopyText
  :text="`题目编号: ${data.questionId}`"
  :copyText="data.questionId"
  @click.stop
/>
```
- **拓展**：修复前先判断改哪一层：改共享层影响面 = 全部调用方，改业务层影响面 = 当前一处。
- *来源：admin-workspace-new 2026-08-11*

### 3. Quill 工具栏按钮统一注册
- **技能点**：学会用「常量 + 联合类型」收敛工具栏按钮的可选范围，让业务方仅凭 toolbar-order 就能用上公式/填空等全部内置按钮。
- **坑点**：此前 formula/image 等按钮的 handler 依赖 customButtons 才注册，业务组件不知道能传哪些名字，不传 customButtons 就用不了。
- **解决方案**：buildToolbarHandlers 对内置按钮无条件注册 handler；新增 toolbarButtons.ts 导出 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 类型，toolbarOrder 类型放宽为 ToolbarButtonName | string。
```text
export const TOOLBAR_BUTTONS = {
  undo: 'undo', redo: 'redo', divider: 'divider',
  formula: 'formula', fillInInsert: 'fillInInsert', image: 'image',
} as const
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[keyof typeof TOOLBAR_BUTTONS]
// 业务侧
toolbarOrder: [B.undo, B.redo, B.divider, B.formula]
```
- **拓展**：凡是要让调用方「按名字选用」的能力，都应配一份导出常量 + 类型，既补全提示又防止拼写漂移。
- *来源：admin-workspace-new 2026-08-11*

### 4. 组件功能做减法：删除多余状态
- **技能点**：实践了「需求收敛时敢于删代码」：把尺寸记忆/持久化/双向同步整链路移除，只留拖拽+折叠+最小宽，组件反而更稳。
- **坑点**：最初为面板做了 storageKey 持久化 + 折叠标志 + sizes 双向同步，复杂度堆叠出递归更新 bug；显隐还用 sizes 里的 'hidden' 字符串这种隐式协议。
- **解决方案**：删掉 storageKey 传参、折叠标志 ref、syncSizesFromFlags 及两个 watch；右栏显隐改用 template v-if 显式控制；保留 ResizablePanels 的 storageKey 能力（空 key 自动跳过）不动共享组件；删完 grep 确认 0 残留引用。
```text
<ResizablePanels v-model:sizes="localPanelSizes">
  <template #panel-0>...</template>
  <template v-if="isAiReviewer" #panel-2>...</template>
</ResizablePanels>
const localPanelSizes = ref([760, 'auto', 400]) // 纯内部拖拽态，不持久化
```
- **拓展**：简化后验证三件事：lint 无新增错误、被删符号零残留、共享组件能力未被破坏。
- *来源：admin-workspace-new 2026-08-11*

### 5. el-splitter 自定义折叠图标
- **技能点**：掌握了「包装组件透传插槽 + 条件检测」的扩展模式：不动底层库行为，只在包装层开口子让业务自定义 UI。
- **坑点**：el-splitter-panel 自带折叠图标样式不符合设计，直接覆盖 CSS 既不灵活也影响其他使用处。
- **解决方案**：ResizablePanels 新增 #start-collapsible 插槽并用 $slots 条件检测透传，隐藏官方图标；业务侧用 svgIcon + CSS rotate(180deg) 区分展开/收起，折叠动作仍由原生面板处理。
```text
<slot v-if="$slots['start-collapsible']" name="start-collapsible" />
<span v-else class="default-collapse-icon" />
/* 业务侧 */
.collapse-icon { transition: transform .2s }
.collapse-icon.collapsed { transform: rotate(180deg) }
```
- **拓展**：包装第三方组件时优先加「命名插槽透传」而不是改源码，升级依赖时零成本。
- *来源：admin-workspace-new 2026-08-10*

### 6. 卡片分隔样式的方案演进
- **技能点**：体会了 UI 细节需多轮对齐：题目分隔从整体边框 → 相邻分隔线（& + &）→ 四周边框+间距，最终用 margin 简写覆盖掉冗余的 first-child 补偿。
- **坑点**：逐版叠加样式容易留下冗余规则（如 margin 简写已含上间距，first-child margin-top 就成了死代码），且宽度占满审核区不符合视觉预期。
- **解决方案**：最终定为 border: 1px solid #ebecf2 + border-radius: 4px + margin: 12px 8px，只保留 :last-child { margin-bottom: 0 }，删掉被简写覆盖的 first-child 规则。
```text
.question-item {
  border: 1px solid #ebecf2;
  border-radius: 4px;
  margin: 12px 8px;
  &:last-child { margin-bottom: 0 }
}
```
- **拓展**：UI 调整收尾时自查一遍：每条声明是否仍被需要、是否已被简写/层叠覆盖。
- *来源：admin-workspace-new 2026-08-11*

### 7. AI 结对编辑的落盘核查
- **技能点**：确认了 AI 辅助编辑的一个真实风险：会话切换可能导致部分 edit 未落盘，磁盘仍是旧版。
- **坑点**：对 toolbarConfig.ts / detailComment.vue 的部分编辑实际未写入文件，基于「以为已改」的后续验证全是空转。
- **解决方案**：关键改动后重新读文件确认磁盘内容，发现为旧版后重新正确应用，再跑 lint 验证 0 错误收口。
- **拓展**：多文件改动收尾时用「读文件 + grep 关键符号」双重确认，别信编辑回执。
- *来源：admin-workspace-new 2026-08-11*

