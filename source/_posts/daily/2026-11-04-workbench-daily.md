---
title: "工作台日报 · 2026-11-04"
date: 2026-11-04 07:01:40
categories: [工作日记]
tags: ["日报", "大模型", "AI", "AI工具", "macOS"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-04

## 🔥 行业热点

- [Postman silently auto-installs AI plugins and attempts to share all API data](https://github.com/postmanlabs/postman-app-support/issues/14370) — *Hacker News*
  - 📌 **内容**：Postman 被指自动安装 AI 插件并尝试共享 API 数据，引发开发者工具隐私与权限边界担忧。
  - 💡 **学习**：使用开发工具时应审计插件安装、权限范围与网络外发行为，关注 API 数据的留存和共享设置。
  - 🧭 **拓展**：用抓包或防火墙监控工具验证其外联请求并检查插件权限。
- [OpenAI seems to be silently re-enabling model training on Codex chats](https://news.ycombinator.com/item?id=50352757) — *Hacker News*
  - 📌 **内容**：有迹象称 OpenAI 在 Codex 聊天中重新启用模型训练，涉及 AI 编程助手的数据使用透明度。
  - 💡 **学习**：关注 AI 编码工具的数据保留、训练开关和隐私设置。
  - 🧭 **拓展**：检查账户隐私选项并对比官方文档与网络请求。
- [Fujifilm to boost tape production 5x as AI fuels archiving demand](https://arstechnica.com/gadgets/2026/10/fujifilm-to-boost-tape-production-5x-as-ai-fuels-archiving-demand/) — *Hacker News*
  - 📌 **内容**：AI 带来的数据归档需求增长推动富士胶片扩大磁带产能，体现冷存储与归档存储的价值。
  - 💡 **学习**：理解磁带存储在低成本长期归档中的定位，以及存储分层思路。
  - 🧭 **拓展**：对比磁带、对象存储与云归档的成本和恢复流程。
- [The Little Book of Generative AI Foundations: An Intuitive Mathematical Primer](https://arxiv.org/abs/2605.29713) — *Hacker News*
  - 📌 **内容**：一本面向直觉理解的生成式 AI 数学基础入门，适合补齐模型背后的数学概念。
  - 💡 **学习**：通过直观材料理解概率、线性代数与生成模型基础。
  - 🧭 **拓展**：结合开源 notebook 复现关键公式与简单生成模型。
- [How I use AI (Oct 2026)](https://ben.stolovitz.com/posts/how-use-ai-oct-2026/) — *Hacker News*
  - 📌 **内容**：作者分享个人使用 AI 的方式，属于 AI 工具融入日常工作流的经验型内容。
  - 💡 **学习**：借鉴 AI 辅助写作、编程与信息整理的工作流设计。
  - 🧭 **拓展**：挑选一两个场景做一周对照实验，记录效率与质量变化。
- [We're Training a Figma-Fluent Editing Model](https://www.figma.com/blog/training-a-figma-fluent-editing-model/) — *Hacker News*
  - 📌 **内容**：团队在训练一个理解 Figma 设计稿并支持编辑的模型，指向设计工具与生成式 AI 的结合。
  - 💡 **学习**：关注设计稿结构化表示、多模态模型与设计到代码的链路。
  - 🧭 **拓展**：用公开 Figma 文件测试设计理解与生成工具的能力边界。
- [Show HN: Kairos – Open-source risk and fraud decisioning platform](https://github.com/finic-ai/kairos) — *Hacker News*
  - 📌 **内容**：Kairos 是开源的风险与欺诈决策平台，面向规则与决策引擎类应用。
  - 💡 **学习**：可学习风控决策平台如何组织规则、特征与决策流程。
  - 🧭 **拓展**：阅读仓库架构并本地部署，用样例数据跑通决策流。
- [The 'Astra can read and write weights to improve Qwen' post is misleading](https://honnibal.dev/blog/astra-read-write-misleading) — *Hacker News*
  - 📌 **内容**：文章质疑“Astra 能读写权重来改进 Qwen”的说法，提醒警惕夸大的模型能力宣传。
  - 💡 **学习**：理解模型权重操作与能力声明的区别，培养技术辨析能力。
  - 🧭 **拓展**：复现相关实验或查阅模型权重修改的可行方法。
- [Qwen 3.8 27B on my laptop can interpret model weights, too](https://simonwillison.net/2026/Nov/3/qwen-can-interpret-model-weights/) — *Hacker News*
  - 📌 **内容**：作者称在笔记本上运行 Qwen 27B 并能解释模型权重，涉及本地大模型与可解释性。
  - 💡 **学习**：了解本地推理部署、模型规模与权重分析工具。
  - 🧭 **拓展**：在本地量化运行小模型并用可视化工具观察权重或激活。
- [Mac at Any Scale](https://namespace.so/blog/mac-at-any-scale) — *Hacker News*
  - 📌 **内容**：围绕 macOS 大规模运行与管理的实践分享，可能涉及 CI、虚拟化或设备集群。
  - 💡 **学习**：了解 macOS 在自动化构建与大规模基础设施中的限制和方案。
  - 🧭 **拓展**：搭建小规模 macOS CI runner 验证虚拟化与编排方案。

## 🌟 GitHub 热门开源项目

- [HKUDS/DeepTutor（LLM 应用）](https://github.com/HKUDS/DeepTutor) — *GitHub · Python · +14.0k/周 · 总 54.7k star*
  - 📌 **是什么**：一个面向终身学习的个性化 AI 辅导系统，结合多智能体、深度检索与 RAG，把教学流程做成可交互的学习产品。
  - 💡 **学习点**：学它如何把多智能体编排、RAG 检索与前端交互串联成完整的学习闭环。
  - 🧭 **上手**：先按 README 跑通最小启动流程，再对照阅读 Agent 与 RAG 模块，看检索结果如何注入对话上下文。
- [cheaper-inference/OmniRoute（开发工具）](https://github.com/cheaper-inference/OmniRoute) — *GitHub · TypeScript · +4.9k/周 · 总 84.3k star*
  - 📌 **是什么**：一个开源 AI 网关，用统一端点聚合多家模型供应商，并兼容 Claude Code、Cursor、Cline 等编码工具。
  - 💡 **学习点**：学 LLM 网关的抽象设计：统一兼容协议、供应商路由、密钥与限流管理，这是前端最容易上手的一层后端能力。
  - 🧭 **上手**：把它配成你常用编码工具的 base URL，抓一次请求日志，观察同一调用如何被路由到不同 provider。
- [AstrBotDevs/AstrBot（Agent 框架）](https://github.com/AstrBotDevs/AstrBot) — *GitHub · Python · +4.5k/周 · 总 45.8k star*
  - 📌 **是什么**：一个集成多种 IM 平台、LLM 与插件生态的聊天机器人框架，支持 MCP，可作为自托管 AI 助手底座。
  - 💡 **学习点**：学插件式架构与 MCP 接入方式，理解消息适配层如何把不同 IM 平台统一成一种事件模型。
  - 🧭 **上手**：用 docker compose 起一个实例并接入一个 IM 平台，然后照着最小插件模板写一个带工具调用的插件。
- [Shubhamsaboo/awesome-llm-apps（LLM 应用）](https://github.com/Shubhamsaboo/awesome-llm-apps) — *GitHub · Python · +3.2k/周 · 总 143.8k star*
  - 📌 **是什么**：收录大量 AI Agent、Agent Skills 与 RAG 应用示例的开源合集，覆盖常见 LLM 应用形态。
  - 💡 **学习点**：通过横向对比不同示例，快速建立“需求到架构选型”的直觉，避免只会一种套路。
  - 🧭 **上手**：挑一个与你手上需求最像的示例完整跑通，再改一处 prompt 或工具做对照实验。
- [browser-use/browser-use（MCP 工具）](https://github.com/browser-use/browser-use) — *GitHub · Python · +2.9k/周 · 总 120.0k star*
  - 📌 **是什么**：让 LLM 驱动浏览器完成任务的 Python 库，配合云端隐身浏览器与托管 Agent API，属于计算机使用类 Agent 基础设施。
  - 💡 **学习点**：学页面可访问性树与 DOM 精简如何转成模型可读的动作空间，这是前端背景最大的优势点。
  - 🧭 **上手**：跑最小示例让它打开一个页面并点击按钮，然后读负责把 DOM 抽成可交互元素索引的那段代码。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步死循环
- **技能点**：掌握双向状态同步（标志 → 数组 → 标志）的正确写法，能在 watch 链路中做值比较保护，避免 deep watch 与引用变化互相唤醒。
- **坑点**：一端 watch 每次都赋新数组、另一端 deep watch 回写标志，即使值未变也会互相触发，报 'Maximum recursive updates exceeded'。
- **解决方案**：每一步先算目标值并做相等比较，相等直接 return，不写回；回写前比较当前值与推导值。
```text
function syncSizesFromFlags() {
  const next = [isFilePreviewFolded.value ? 0 : 760, 'auto', rightWidth.value]
  if (next[0] === localPanelSizes.value[0] && next[2] === localPanelSizes.value[2]) return
  localPanelSizes.value = next
}
watch(localPanelSizes, (val) => {
  const folded = val[0] === 0
  if (isFilePreviewFolded.value !== folded) isFilePreviewFolded.value = folded
}, { deep: true })
```
- **拓展**：能推广到 URL query ↔ 组件状态、表单 ↔ 查询条件等所有双向同步场景；必要时改用单向数据流加 computed。
- *来源：admin-workspace-new / 2026-08-11*

### 2. 事件冒泡修复位置的选择
- **技能点**：判断交互 bug 该改共享组件还是业务调用点，用最小可用 diff 解决冒泡问题。
- **坑点**：共享 CopyText 内部点击冒泡到外层卡片的 @click，点复制顺带触发展开/收起；若在共享组件里 stopPropagation，会污染所有使用方。
- **解决方案**：共享组件保持通用、回退改动，在业务调用点加 @click.stop，并通过 prop 区分「显示文案」与「实际复制内容」。
```text
<CopyText
  :text="`题目编号: ${data.questionId}`"
  :copyText="data.questionId"
  @click.stop
/>
```
- **拓展**：形成约定：通用组件的默认行为不改，调用点按需阻断；组件库层只提供能力，不替业务做决定。
- *来源：admin-workspace-new / 2026-08-11*

### 3. 组件能力注册表化
- **技能点**：把组件的可选配置项收敛为导出的常量数组加联合类型，让调用方有类型提示和单一事实来源。
- **坑点**：业务方不知道 toolbar-order 能传哪些名字，特殊按钮还必须另配 customButtons 才生效；名字靠字符串硬编码，改名无从发现。
- **解决方案**：导出 TOOLBAR_BUTTONS as const 与 ToolbarButtonName 联合类型，内置按钮统一无条件注册，调用方用常量引用，props 类型写成 ToolbarButtonName | string 以兼容扩展。
```text
export const TOOLBAR_BUTTONS = ['undo', 'redo', 'divider', 'fillInInsert', 'formula', 'image'] as const
export type ToolbarButtonName = (typeof TOOLBAR_BUTTONS)[number]

props: { toolbarOrder: { type: Array as PropType<(ToolbarButtonName | string)[]> } }
```
- **拓展**：可复用到所有枚举型 props（列配置、图表类型、按钮组）；进一步用类型守卫或运行时校验拦截拼错的名字。
- *来源：admin-workspace-new / 2026-08-11*

### 4. 重构分支合并主干的策略
- **技能点**：在大规模格式化与主干合并同时存在时，选择合适的合并路径控制冲突规模。
- **坑点**：把主干也全量 prettier 后再 merge（镜像格式化分支），会让「仅我方改」的文件变成双方都改、行布局错位，冲突从 51 文件/118 块放大到 117 文件/414 块。
- **解决方案**：直接 git merge 主干 → 冲突处整体采纳主干版 → 统一跑 format → 按清单 grep 驱动地在主干基线上重做语义成果。
```text
git merge origin/master
# 冲突处整体采纳主干版后统一排版
npm run format
grep -rn "目标符号" src/
```
- **拓展**：合并前先统计与主干的文件差异清单，按「仅我方改 / 双方改」分类，分批合入；结论沉淀进合并实施文档。
- *来源：admin-workspace-hr / 2026-10-14*

### 5. 删改前的影响面核对
- **技能点**：改动前用全仓搜索建立引用清单，删除后核对无残留，并对同一文件的多处编辑逐条顺序执行。
- **坑点**：直接删符号容易留下悬挂引用；同一文件在一次操作中并行编辑会基于同一份快照互相覆盖，出现「编辑未落盘」。
- **解决方案**：先 grep 全引用列出影响清单再动手，删除后用同一组关键词复查 0 残留；同一文件的编辑逐条顺序执行，并回读磁盘确认生效。
```text
grep -rn "storageKey\|syncSizesFromFlags\|isFilePreviewFolded" src/
# 删除后复查命中数为 0 再提交
```
- **拓展**：可封装成删除/重命名前的脚本化预检；会话切换或长任务后强制回读磁盘，避免基于过期快照判断。
- *来源：admin-workspace-new / 2026-08-11*

### 6. 验证基线区分存量与新引入
- **技能点**：用类型检查加 lint 的基线对比判断本次改动是否引入新问题，而不是被存量错误干扰。
- **坑点**：仓库本身有大量预存的 any 类型错误，跑完整构建既慢又噪声大，容易误判为本次改动引入。
- **解决方案**：默认只跑类型检查，lint 只关注新增条目；存量错误记录在案不顺手修，涉及构建配置或依赖时才跑完整构建。
```text
npx tsc --noEmit
# 仅本次改动文件：确认 lint ERROR 均为预存项，新增为 0
```
- **拓展**：把存量错误清单固化成基线文件，配合增量检查脚本只对新改动报错，逐步清零。
- *来源：admin-workspace-hr / 2026-10-15*

