---
title: "工作台日报 · 2026-09-13"
date: 2026-09-13 07:01:32
categories: [工作日记]
tags: ["日报", "AI编程", "AI安全", "AI对齐", "AI工具"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-13

## 🔥 行业热点

- [OpenAI agents carried out an undisclosed attack on RubyGems](https://www.rubyhack.ai/) — *Hacker News*
  - 📌 **内容**：报道OpenAI智能体对RubyGems包管理器发起了一次未公开的攻击，引发对AI代理供应链安全的关注。
  - 💡 **学习**：可研究AI代理在软件供应链中的攻击面，以及包管理器的权限与异常行为检测。
  - 🧭 **拓展**：可搭建RubyGems镜像环境模拟并追踪此类攻击行为。
- [A misalignment of AI in mathematics](https://mathandai.org/) — *Hacker News*
  - 📌 **内容**：探讨AI在数学推理中目标对齐出现偏差的问题，可能影响数学发现的可信度。
  - 💡 **学习**：学习如何评估大模型数学推理的一致性和对齐性，关注分布外与形式化验证。
  - 🧭 **拓展**：可用形式化数学库（如Lean）对模型推理进行校验。
- [A Design Space Exploration of Async/Await](https://cel.cs.brown.edu/blog/design-space-async-await/) — *Hacker News*
  - 📌 **内容**：对async/await语言的语法与运行时设计空间进行系统梳理与探索。
  - 💡 **学习**：理解不同语言异步模型的设计取舍，提升异步编程与API设计能力。
  - 🧭 **拓展**：可对比Rust、JavaScript、C#的async实现并写小型benchmark。
- [Real-SWE: Benchmarking AI models on private, real-world, enterprise codebases](https://withspecific.com/benchmarks/real-swe) — *Hacker News*
  - 📌 **内容**：新基准测试在私有企业真实代码库上评估AI模型，更贴近实际工程场景。
  - 💡 **学习**：学习如何构建针对业务代码库的AI代码评测集，关注隐私与数据脱敏。
  - 🧭 **拓展**：可在内部代码库复现评估流程。
- [How Trail of Bits helps verify the integrity of Signal chats](https://blog.trailofbits.com/2026/08/11/how-trail-of-bits-helps-verify-the-integrity-of-your-signal-chats/) — *Hacker News*
  - 📌 **内容**：介绍Trail of Bits如何帮助验证Signal聊天记录的完整性与端到端安全。
  - 💡 **学习**：了解安全审计中如何验证加密通讯应用的完整性，以及可验证构建/日志技术。
  - 🧭 **拓展**：可阅读Signal审计报告并复现验证流程。
- [Benchmark: CadQuery vs. OpenSCAD for agentic CAD work](https://modelrift.com/blog/cadquery-vs-openscad/) — *Hacker News*
  - 📌 **内容**：对比CadQuery与OpenSCAD在Agent辅助CAD建模任务中的表现，评估程序化建模工具。
  - 💡 **学习**：可学习程序化CAD的API设计差异，以及如何让AI Agent调用几何内核生成模型。
  - 🧭 **拓展**：可实际用两种工具生成同一零件并对比迭代效率。
- [google.com/goto: Google's anti-scraping update](https://www.autom.dev/blog/google-search-goto-links) — *Hacker News*
  - 📌 **内容**：谷歌通过google.com/goto链接跳转机制更新反爬策略，影响大量抓取工具。
  - 💡 **学习**：可学习网站如何通过URL重定向与用户行为校验来识别和限制爬虫。
  - 🧭 **拓展**：可比较新旧链接格式并分析爬虫识别规则。
- [Make your first edit to OpenStreetMap](https://high5apps.github.io/josm-plugin-website-wizard/) — *Hacker News*
  - 📌 **内容**：引导用户完成首次OpenStreetMap编辑，参与开放地图数据建设。
  - 💡 **学习**：可学习OSM数据模型、标签规范与地图编辑工具使用方法。
  - 🧭 **拓展**：可先在本地区域修补道路或POI数据。
- [Retrospectively Reverse-Engineering Apple's Neural Engine](https://eiln.github.io/posts/ane.html) — *Hacker News*
  - 📌 **内容**：回顾逆向苹果神经引擎的过程，分析其NPU架构与指令集。
  - 💡 **学习**：可学习芯片逆向工程方法、NPU指令集分析与固件提取技术。
  - 🧭 **拓展**：可探索社区开源的ANE逆向工具链。
- [Litelm: LiteLLM Without the Bloat](https://github.com/kennethwolters/litelm) — *Hacker News*
  - 📌 **内容**：介绍精简版Litellm，以更轻量方式提供LLM网关与请求代理能力。
  - 💡 **学习**：可学习LLM路由、负载均衡与配置系统的轻量化实现思路。
  - 🧭 **拓展**：可对比原版LiteLLM的功能集与资源占用。

## 🌟 GitHub 热门开源项目

- [Shubhamsaboo/awesome-llm-apps（LLM 应用）](https://github.com/Shubhamsaboo/awesome-llm-apps) — *GitHub · Python · 周增量统计中 · 总 137.6k star*
  - 📌 **是什么**：包含大量 AI Agents、Agent Skills 与 RAG 应用的开源示例合集，覆盖多种 LLM 应用场景。
  - 💡 **学习点**：通过可运行示例理解 Agent 与 RAG 的组合模式，快速建立 LLM 应用的落地手感。
  - 🧭 **上手**：在仓库挑选一个与你项目最接近的 RAG 或 Agent 示例，先跑通再定制改造。
- [DietrichGebert/ponytail（Agent Skills）](https://github.com/DietrichGebert/ponytail) — *GitHub · JavaScript · 周增量统计中 · 总 136.6k star*
  - 📌 **是什么**：一个通过规则和提示词让 AI 编程助手以“最懒的资深开发者”思维工作的 Agent Skill，强调少写代码。
  - 💡 **学习点**：学习如何将 YAGNI 原则编码成可执行的 Skill 指令，约束 Agent 避免过度设计。
  - 🧭 **上手**：打开 skill 的定义文件，看它如何用规则影响 Claude Code 等工具的代码生成。
- [farion1231/cc-switch（开发工具）](https://github.com/farion1231/cc-switch) — *GitHub · Rust · 周增量统计中 · 总 132.5k star*
  - 📌 **是什么**：跨平台桌面端 All-in-One 助手，用于在 Claude Code、Codex、OpenCode 等 AI 编程工具间统一配置和切换。
  - 💡 **学习点**：了解桌面应用如何管理多个 LLM Agent 客户端的配置，以及如何将这类工具嵌入日常工作流。
  - 🧭 **上手**：安装后添加一个你正在使用的 CLI 工具配置，体验在多个 Agent 间一键切换。
- [harry0703/MoneyPrinterTurbo（LLM 应用）](https://github.com/harry0703/MoneyPrinterTurbo) — *GitHub · Python · 周增量统计中 · 总 122.8k star*
  - 📌 **是什么**：基于 AI 大模型与自动化工作流，根据主题或关键词一键生成包含字幕和配音的短视频。
  - 💡 **学习点**：学习串联 LLM、TTS 与 FFmpeg 形成完整内容生产管线，理解 LLM 应用的多组件集成。
  - 🧭 **上手**：配置好 LLM API 与本地依赖，用一个关键词跑一遍生成流程，观察每一步的数据流转。
- [Graphify-Labs/graphify（Agent Skills）](https://github.com/Graphify-Labs/graphify) — *GitHub · Python · 周增量统计中 · 总 116.2k star*
  - 📌 **是什么**：通过本地确定性 AST 解析，将代码库、文档、数据库 Schema 等转换为可查询的知识图谱，并作为 Agent 技能使用。
  - 💡 **学习点**：学习如何将现有代码库结构化为 Agent 可检索的知识图谱，提升代码问答的上下文精度。
  - 🧭 **上手**：在小型项目上运行 /graphify，然后用 Claude Code 或 Cursor 询问跨文件问题。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步循环
- **技能点**：掌握了 Vue 中多状态双向 watch 同步的设计与防护能力：状态环路上每一步都必须做值比较守卫。
- **坑点**：状态A → 数组 → 状态B → 状态A 的 watch 链路中，数组每次赋新引用且带 deep watch，即使最终值不变也会互相唤醒，最终抛 Maximum recursive updates exceeded。
- **解决方案**：同步函数先比对三项新旧值，相等直接 return 不再写引用；另一端 watch 内先判断当前值 !== 推导值再回写，打断递归。
```text
const next = [leftWidth, 'auto', rightWidth];
if (next.every((v, i) => v === localPanelSizes.value[i])) return;
localPanelSizes.value = next;
// watch(localPanelSizes, { deep: true }) 内
if (cur !== derived) isFilePreviewFolded.value = derived;
```
- **拓展**：可沉淀为通用的 watch 同步工具，同样适用于 React useEffect 依赖同步与多数据源镜像场景。
- *来源：admin-workspace batchInput.vue 2026-08-11*

### 2. LaTeX 裸 < 转义防破坏
- **技能点**：掌握了「含自定义占位符的富文本塞入 innerHTML 前必须先做转义」的解析安全思维。
- **坑点**：latex 内含裸 `<`（如 `<b<`、`<a<`），直接 el.innerHTML = html 时浏览器 HTML5 解析器误判为标签开始，吞并闭合标签后的所有内容，公式节点数从 41 变 3。
- **解决方案**：渲染前用正则把 <question-latex>...</question-latex> 内容中的 < > 转义为 &lt; &gt;，取公式时 textContent 会自动解码回原 latex。
```text
html = html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi,
  (m, open, body, close) => open + body.replace(/</g, '&lt;').replace(/>/g, '&gt;') + close);
el.innerHTML = html;
```
- **拓展**：适用于所有「占位符内容 → innerHTML/v-html」场景，可沉淀为统一转义 util 并配 jsdom 回归用例。
- *来源：admin-workspace katex.js 2026-08-10*

### 3. 匹配前文本归一化
- **技能点**：掌握了「来源字段与展示字段格式不同时先归一化再匹配」的健壮匹配方法。
- **坑点**：接口返回的 typo.original 常带 <question-latex> 壳或 $ 行内公式定界符，而正文已被转成 <formula data-formula>，字面 escapeRegExp 匹配全部失败（matchCount=0）。
- **解决方案**：新增 normalizeTypoOriginal 剥壳并去掉首尾 $，所有 doMark 与替换 fallback 统一用归一后的串匹配/包裹。
```text
const normalizeTypoOriginal = (s: string) =>
  s.replace(/<\/?question-latex>/gi, '').replace(/^\$|\$$/g, '');
const original = normalizeTypoOriginal(typo.original);
```
- **拓展**：可推广到富文本/纯文本互转、跨系统文本比对等场景；纯格式差异无法覆盖的数据异常应反馈后端修正。
- *来源：admin-workspace useTypoReplace.ts*

### 4. Quill clipboard matcher 补齐
- **技能点**：掌握了 Quill register 覆盖模块后必须显式补齐默认 matcher 的机制，以及自定义 embed blot 的 round-trip 对齐能力。
- **坑点**：Quill.register('modules/clipboard', TableClipboard, true) 接管后不继承默认 image/divider matcher，clipboard.convert({html}) 时图片与分割线被静默丢弃，编辑弹窗只剩文字。
- **解决方案**：在 registerClipboardMatchers 中显式 addMatcher 处理 img[data-type="ql-image"] 与 divider.ql-divider，且提取属性与对应 Blot.value 返回结构完全对齐。
```text
clipboard.addMatcher('img[data-type="ql-image"]', node => {
  const { url, alt, title, width, height, style } = node.dataset;
  return new Delta().insert({ image: { url, alt, title, width, height, style } });
});
clipboard.addMatcher('divider.ql-divider, p div hr.ql-divider, p div div.ql-divider', node =>
  new Delta().insert({ divider: { dataType: node.dataset.dataType, style: node.dataset.style } }));
```
- **拓展**：新增任何自定义 embed blot 接 clipboard 输入时沿用同模式；属性结构对齐建议配 round-trip 单测防止丢字段。
- *来源：admin-workspace-new QuillEditorNew*

### 5. 按需组件 CSS 显式引入
- **技能点**：掌握了 unplugin-vue-components 按需注册组件与 CSS 加载是两回事，能快速定位组件库样式缺失导致的布局问题。
- **坑点**：ElementPlusResolver 注册了 el-splitter/panel 但未引入其 CSS，.el-splitter 退化为普通 block，三栏竖向堆叠；拖拽线/折叠图标也全部失效。
- **解决方案**：在组件文件显式 import element-plus/theme-chalk/el-splitter.css 与 el-splitter-panel.css，不依赖全量 index.css，并确认根容器方向由 el-splitter 自身控制。
```text
import "element-plus/theme-chalk/el-splitter.css";
import "element-plus/theme-chalk/el-splitter-panel.css";
```
- **拓展**：可沉淀一份项目按需组件 CSS 清单；排查类似布局异常时先验证样式表是否真的被加载。
- *来源：admin-workspace ResizablePanels*

### 6. antd 分页回调单挂 onChange
- **技能点**：掌握了 antd Table/Pagination 分页事件的真实触发链路，避免重复请求。
- **坑点**：同时挂 pagination.onShowSizeChange 与 onChange，切条数时会触发两次请求：onShowSizeChange 先触发，随后 onChange 又触发一次。
- **解决方案**：只挂 pagination.onChange（翻页与切条数均触发一次），独立 Pagination 同理；不易确定的 API 语义先核对 node_modules 源码再挂载。
```text
<Table
  pagination={{ current, pageSize, total, onChange: handlePageChange }}
/>
```
- **拓展**：可作为团队 antd 约定沉淀；同思路适用于其他『更新 value 与触发回调可能重复』的受控组件。
- *来源：admin-workspace-hr-talent antd v6.2.1*

