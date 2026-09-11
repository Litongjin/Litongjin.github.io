---
title: "工作台日报 · 2026-09-11"
date: 2026-09-11 18:32:46
categories: [工作日记]
tags: ["日报", "AI工具", "大模型", "AI编程", "Agent"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-11

## 🔥 行业热点

- [OpenAI Agents API](https://developers.openai.com/api/docs/guides/agents-api/overview) — *Hacker News*
  - 📌 **内容**：围绕 OpenAI Agents API 的发布或文档，开发者可以用它构建更自动化的智能体工作流。
  - 💡 **学习**：可学习 Agent 编排、工具调用与上下文管理的最佳实践。
  - 🧭 **拓展**：用一个小型自动化脚本验证 API 的稳定性和成本。
- [Astra for Coding: Why Are We Doing This Again?](https://lucumr.pocoo.org/2026/9/7/astra-why/) — *Hacker News*
  - 📌 **内容**：一篇对“Astra”用于编程的反思性文章，质疑不断重复制造 AI 编程工具的现象。
  - 💡 **学习**：对比不同 AI 编程助手时，应关注实际开发流程而非盲目追逐新工具。
  - 🧭 **拓展**：可围绕现有项目做一次 AI 编码工具的横向评测。
- [OpenAI’s Navier-Stokes release included a Lean 4 formal proof](https://www.johndcook.com/blog/2026/09/09/formal-method-revolution/) — *Hacker News*
  - 📌 **内容**：OpenAI 发布与纳维-斯托克斯相关的内容，并附带 Lean 4 形式化证明，体现 AI 与定理证明结合。
  - 💡 **学习**：可以用 Lean 4 把数学或代码的关键性质形式化，减少歧义。
  - 🧭 **拓展**：尝试用 Lean 4 验证一个简单算法的不变量。
- [Compute-efficient pretraining and scaling to trillion-parameter models](https://magic.dev/blog/pretraining#) — *Hacker News*
  - 📌 **内容**：讨论计算效率更高的预训练方法，以及将模型扩展到万亿参数级别的路径。
  - 💡 **学习**：了解数据配比、调度策略等计算效率优化，而非只堆参数。
  - 🧭 **拓展**：可复现小规模实验，验证 scaling law 在自己的任务上是否成立。
- [The Gemini app is now available for Windows](https://blog.google/innovation-and-ai/products/gemini-app/gemini-app-now-on-windows/) — *Hacker News*
  - 📌 **内容**：Gemini 应用正式登陆 Windows，用户可在桌面端直接使用 AI 助手。
  - 💡 **学习**：关注桌面端 AI 应用的集成方式和本地调用能力。
  - 🧭 **拓展**：可安装体验，对比 Web 端和桌面端的使用差异。
- [Shopify is moving from React Native back to Swift and Kotlin](https://shopify.engineering/back-to-native) — *Hacker News*
  - 📌 **内容**：Shopify 决定从 React Native 迁移回原生 Swift 和 Kotlin，反映跨平台框架在大型 App 上的取舍。
  - 💡 **学习**：评估跨端方案时需要考虑团队、性能和原生体验的长期成本。
  - 🧭 **拓展**：可分析自身 App 中跨端与原生混编的边界。
- [DeepSeek v4.1 Flash](https://twitter.com/deepseek_ai/status/2097930608790167907) — *Hacker News*
  - 📌 **内容**：DeepSeek 发布 v4.1 Flash 模型，主打轻量高效的推理能力。
  - 💡 **学习**：可对比 Flash 版与标准版在延迟、成本和效果上的差异。
  - 🧭 **拓展**：在 API 场景中测试 Flash 模型对常见任务的支持表现。
- [Rust is tier-1 language at Microsoft](https://rustfoundation.org/media/guest-post-rust-is-tier-1-language-at-microsoft/) — *Hacker News*
  - 📌 **内容**：微软将 Rust 提升为一等语言，系统编程在微软生态中的地位继续加强。
  - 💡 **学习**：可学习 Rust 的所有权与内存安全模型，用于底层组件开发。
  - 🧭 **拓展**：尝试用 Rust 写一个 CLI 工具并接入 Windows API。
- [Cognition launches new SWE-2 model, Rivaling Fable 5.1 and GPT-Astra](https://cognition.com/blog/swe-2) — *Hacker News*
  - 📌 **内容**：Cognition 发布新一代 SWE-2 模型，目标是在软件工程任务上与 Fable 5.1、GPT-Astra 竞争。
  - 💡 **学习**：关注 Agent 在真实代码仓库上的评测指标，如 issue 解决率。
  - 🧭 **拓展**：用 SWE-2 跑一个开源 issue 验证其自动化修 bug 能力。
- [Neki – Sharded Postgres](https://planetscale.com/blog/introducing-neki) — *Hacker News*
  - 📌 **内容**：Neki 是一个对 Postgres 进行分片扩展的方案或产品，旨在突破单库瓶颈。
  - 💡 **学习**：了解 Postgres 分片键设计、数据分布与查询路由的权衡。
  - 🧭 **拓展**：可在测试环境模拟多分片集群，观察跨分片查询性能。

## 🌟 GitHub 热门开源项目

- [Significant-Gravitas/AutoGPT（Agent 框架）](https://github.com/Significant-Gravitas/AutoGPT) — *GitHub · Python · 周增量统计中 · 总 187.3k star*
  - 📌 **是什么**：一个让每个人都能使用和构建 AI 的开源自主代理项目，提供多步骤任务规划与执行能力。
  - 💡 **学习点**：学习用 Python 搭建自主 Agent 的核心循环：目标分解、工具调用与结果反馈。
  - 🧭 **上手**：阅读 README 的快速开始，跑通一个最简单的“目标→计划→执行”示例。
- [firecrawl/firecrawl（MCP 工具）](https://github.com/firecrawl/firecrawl) — *GitHub · TypeScript · 周增量统计中 · 总 179.0k star*
  - 📌 **是什么**：面向 AI 的网页搜索、抓取与交互 API，可将网页内容转为适合 LLM 的 Markdown。
  - 💡 **学习点**：学习如何用 TypeScript 封装爬虫工具，并设计成 AI 可直接调用的 Web API。
  - 🧭 **上手**：查看 API 示例，用本地网页跑一次 HTML 转 Markdown，观察输出质量。
- [langgenius/dify（Agent 框架）](https://github.com/langgenius/dify) — *GitHub · TypeScript · 周增量统计中 · 总 155.4k star*
  - 📌 **是什么**：一个低代码平台，用于构建 Agentic 工作流和 RAG 管道，支持多模型与工具，可云或自托管部署。
  - 💡 **学习点**：学习可视化编排 Agent 工作流的方式，理解节点、工具与知识库如何协同。
  - 🧭 **上手**：在官方 Demo 中拖拽一个简单对话流程，观察不同节点如何串联。
- [langflow-ai/langflow（Agent 框架）](https://github.com/langflow-ai/langflow) — *GitHub · Python · 周增量统计中 · 总 154.6k star*
  - 📌 **是什么**：一个可视化构建和部署 AI 代理与工作流的低代码工具，基于 Python 生态。
  - 💡 **学习点**：学习用 React Flow 等前端技术构建流程编辑器，掌握 Agent 工作流的交互设计。
  - 🧭 **上手**：启动本地 demo，拖拽连接一个 LLM 和工具节点，生成并运行一个简单 agent。
- [langchain-ai/langchain（Agent 框架）](https://github.com/langchain-ai/langchain) — *GitHub · Python · 周增量统计中 · 总 146.1k star*
  - 📌 **是什么**：一个 Agent 工程平台，提供构建、运营 AI 代理的完整工具链和抽象。
  - 💡 **学习点**：学习 LangChain 的核心抽象（Chain、Tool、Agent），理解 LLM 与外部工具的集成模式。
  - 🧭 **上手**：按照官方 Quickstart 教程，跑通第一个 LLM 调用链。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步递归
- **技能点**：掌握 Vue 响应式派生/同步的循环根因，养成每步写值比较守卫的防御性编程习惯。
- **坑点**：两个 watch 相互触发（折叠标志 ↔ panel sizes 数组），即使值未变化，数组新引用和 deep watch 也导致无限递归，最终报 Maximum recursive updates exceeded。
- **解决方案**：syncSizesFromFlags 计算 next 后逐项比对，与当前值相同则直接 return；watch 回写标志前也先判断 当前值 !== 推导值。
```text
const next: PanelSizes = [leftWidth, 'auto', rightWidth];
const same = next.every((v, i) => v === localPanelSizes.value[i]);
if (same) return;
localPanelSizes.value = next;

// 另一 watch
if (isFilePreviewFolded.value !== nextFolded) isFilePreviewFolded.value = nextFolded;
```
- **拓展**：可延伸至 React useEffect/Redux 等派生状态同步；更优做法是单一数据源 + 单向派生，彻底消除双向 watch。
- *来源：admin-workspace-new | 2026-08-11*

### 2. 按需注册组件但 CSS 未引入
- **技能点**：学会排查按需加载组件库样式失效：按需注册组件 ≠ 样式自动加载，需显式引入对应 CSS。
- **坑点**：使用 unplugin-vue-components + ElementPlusResolver 按需注册时，el-splitter/el-splitter-panel 这类较新组件的 CSS 未自动补齐，导致组件退化为块级布局、三栏堆叠。
- **解决方案**：在组件文件显式 import element-plus 的 el-splitter.css 和 el-splitter-panel.css；排查时优先确认对应 CSS 文件是否真的被项目引入。
```text
import "element-plus/theme-chalk/el-splitter.css";
import "element-plus/theme-chalk/el-splitter-panel.css";
```
- **拓展**：适用于所有按需加载 UI 库（antd、Vant 等），可建立 组件类型→按需 CSS 清单 的检查表或工具脚本。
- *来源：admin-workspace | MEMORY.md*

### 3. innerHTML 解析破坏裸 < 标签
- **技能点**：理解浏览器 HTML5 解析器对 innerHTML 的宽容处理，掌握在插入动态内容前对尖括号做转义或使用文本节点。
- **坑点**：直接 el.innerHTML = html 时，LaTeX 内容中的裸 <（如 <b<）被误判为标签开始，破坏 <question-latex> 闭合，后续内容全部并入同一公式。
- **解决方案**：在赋值前用正则将 <question-latex>...</question-latex> 内部的 < > 转义为 &lt; &gt;；读取时用 textContent 自动解码。
```text
const safe = html.replace(
  /(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi,
  (_, open, body, close) =>
    open + body.replace(/</g, '&lt;').replace(/>/g, '&gt;') + close
);
el.innerHTML = safe;
```
- **拓展**：可作为通用富文本清洗函数沉淀；同理适用于渲染用户输入时做 HTML 转义，防 XSS 与结构破坏。
- *来源：admin-workspace | MEMORY.md*

### 4. 受控组件 value 的 falsy 陷阱
- **技能点**：掌握受控组件与 || undefined 结合时假值丢失的坑，学会用字符串/显式判断保住 0 等合法值。
- **坑点**：数字 0 被 value={x || undefined} 当 falsy 丢掉，导致“第0轮”等选项无法回显、状态混乱。
- **解决方案**：对以 0 开始的选项 value 使用字符串 '0'，onChange 时用 Number() 转回；或统一用 x === undefined ? undefined : x。
```text
<Select
  value={round === undefined ? undefined : String(round)}
  onChange={(v) => onChange(v === undefined ? undefined : Number(v))}
>
```
- **拓展**：可封装 normUndef 工具函数并推广到所有受控组件；也适用于 Vue 的 v-model 绑定。
- *来源：admin-workspace-hr | MEMORY.md*

### 5. Table 表头吸顶与滚动错位
- **技能点**：掌握 antd Table sticky 与 scroll.x 的配合关系，以及用 CSS sticky 实现 表头吸顶 + 仅纵向滚动 的正确姿势。
- **坑点**：单独启用 sticky（尤其 scroll.x 未配置或列宽不足）会导致表头与 body 右侧列错位，且 sticky 自带横向滚动条出现多余空白。
- **解决方案**：需要横向滚动时 sticky 必须配 scroll={{ x: 'max-content' }}；仅吸顶时不用 sticky，改为对 thead th 加 position: sticky; top: 0; z-index: 2。
```text
.xxx-table .ant-table-thead > tr > th {
  position: sticky;
  top: 0;
  z-index: 2;
}
```
- **拓展**：可沉淀为通用 StickyTable 封装；同时可推广到其他基于滚动容器的组件。
- *来源：admin-workspace-hr-talent | MEMORY.md*

### 6. 数据归一化后再匹配
- **技能点**：学会在比对/替换前将不同来源的数据归一化（剥离包装、定界符），避免字面匹配失败。
- **坑点**：接口返回的 original 常带 <question-latex> 壳或 $ 定界符，而正文已被转成 <formula data-formula>，直接 escapeRegExp(original) 匹配全部失败（matchCount=0）。
- **解决方案**：新增 normalizeTypoOriginal 剥离标签壳与首尾 $，在标记/替换逻辑中统一使用归一化后的字符串进行匹配。
```text
const normalize = (s: string) =>
  s.replace(/<\/?question-latex>/gi, '').replace(/^\$+|\$+$/g, '');

const hit = text.indexOf(normalize(original));
```
- **拓展**：可用于富文本、LaTeX、Markdown 等混合格式的匹配；后续建议后端统一返回无壳数据，前端归一化作为防御。
- *来源：admin-workspace | MEMORY.md*

