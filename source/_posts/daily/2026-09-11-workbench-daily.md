---
title: "工作台日报 · 2026-09-11"
date: 2026-09-11 07:01:48
categories: [工作日记]
tags: ["日报", "大模型", "AI安全", "AI工具", "AI模型"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-11

## 🔥 行业热点

- [OpenAI Agents API](https://developers.openai.com/api/docs/guides/agents-api/overview) — *Hacker News*
  - 📌 **内容**：OpenAI 正式推出 Agents API，面向开发者提供构建自主 Agent 的接口与工具链。
  - 💡 **学习**：可学习 Agent 工作流、工具调用与任务规划的标准化实现方式。
  - 🧭 **拓展**：可基于该 API 搭建一个具备搜索或代码执行能力的个人助手原型。
- [More questions about whether researchers can trust OpenAI with unpublished math](https://mathstodon.xyz/@andreasthom/117240535270608201) — *Hacker News*
  - 📌 **内容**：学界对将未发表数学成果交给 OpenAI 提出更多信任质疑，涉及数据使用与研究伦理。
  - 💡 **学习**：开发者和研究者需关注第三方 AI 服务的数据使用边界与保密承诺。
  - 🧭 **拓展**：可在合作前审查模型提供方的隐私条款、数据保留与审计策略。
- [Compute-efficient pretraining and scaling to trillion-parameter models](https://magic.dev/blog/pretraining#) — *Hacker News*
  - 📌 **内容**：围绕如何以高效算力预训练万亿参数模型展开讨论，聚焦训练成本与扩展瓶颈。
  - 💡 **学习**：可学习稀疏激活、数据配比、优化器并行等降低大规模训练成本的方法。
  - 🧭 **拓展**：可复现论文中的 scaling law 公式，在小规模实验上验证结论。
- [OpenAI’s Navier-Stokes release included a Lean 4 formal proof](https://www.johndcook.com/blog/2026/09/09/formal-method-revolution/) — *Hacker News*
  - 📌 **内容**：OpenAI 在 Navier-Stokes 相关发布中包含了 Lean 4 形式化证明，展示 AI 与数学证明结合的新进展。
  - 💡 **学习**：可了解 Lean 4 证明助手如何用于验证复杂科学或数学结论。
  - 🧭 **拓展**：可尝试用 Lean 4 写出简单定理的证明，体验形式化验证流程。
- [Detecting and countering misuse of AI: September 2026](https://www.anthropic.com/threat-intelligence-report-september-2026) — *Hacker News*
  - 📌 **内容**：一份关于 2026 年 AI 滥用检测与应对的报告，总结对抗攻击、异常识别与防护手段。
  - 💡 **学习**：可学习红队测试、滥用监控与 AI 安全防护的落地方法。
  - 🧭 **拓展**：可为自己的 AI 应用设计一套 misuse 检测与响应流程。
- [DeepSeek v4.1 Flash](https://twitter.com/deepseek_ai/status/2097930608790167907) — *Hacker News*
  - 📌 **内容**：DeepSeek 发布 v4.1 Flash 模型，主打更轻量高效的大模型推理能力。
  - 💡 **学习**：可关注轻量模型在推理成本、延迟与本地部署上的优化思路。
  - 🧭 **拓展**：可在代码生成或问答任务上评测其与同类模型的差异。
- [Shopify is moving from React Native back to Swift and Kotlin](https://shopify.engineering/back-to-native) — *Hacker News*
  - 📌 **内容**：Shopify 宣布从 React Native 回归原生 Swift/Kotlin，重新权衡跨平台与原生体验。
  - 💡 **学习**：可学习跨平台框架选型时对性能、维护成本与团队结构的综合评估。
  - 🧭 **拓展**：可调研其他大型应用在跨平台与原生方案之间迁移的案例。
- [Rust is tier-1 language at Microsoft](https://rustfoundation.org/media/guest-post-rust-is-tier-1-language-at-microsoft/) — *Hacker News*
  - 📌 **内容**：Rust 在微软被提升为 tier-1 语言，意味着获得官方工具链与平台支持。
  - 💡 **学习**：可关注微软对 Rust 生态的投资，以及 Rust 在系统编程中的最佳实践。
  - 🧭 **拓展**：可尝试用 Rust 编写一个 Windows 平台小工具或 CLI 应用。
- [What algorithm did Windows XP use to choose your initial user picture?](https://devblogs.microsoft.com/oldnewthing/20260909-00/?p=112683) — *Hacker News*
  - 📌 **内容**：文章探讨 Windows XP 初始用户头像选择背后的算法，属于系统历史与逆向工程话题。
  - 💡 **学习**：可从中理解旧系统 UI 逻辑以及简单随机或哈希选择算法的设计。
  - 🧭 **拓展**：可逆向或复现该规则，验证不同安装序列下的头像分布。
- [Cognition launches new SWE-2 model, Rivaling Fable 5.1 and GPT-Astra](https://cognition.com/blog/swe-2) — *Hacker News*
  - 📌 **内容**：Cognition 发布 SWE-2 模型，宣称在软件工程任务上可与 Fable 5.1、GPT-Astra 竞争。
  - 💡 **学习**：可关注面向编程软件工程任务的模型能力评估与基准测试方法。
  - 🧭 **拓展**：可在 SWE-bench 等基准上测试该模型的实际表现。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步循环
- **技能点**：掌握 Vue 中多状态与数组双向同步的循环防护，养成写回前比较值的习惯。
- **坑点**：状态A→数组→watch(deep)→回写状态B→watch状态B→再写数组，数组引用变化触发无限循环。
- **解决方案**：在每处写回前比较当前值与推导值，相等直接 return；让双向同步的 watch 均有守卫。
```text
const next = syncSizesFromFlags();
if (JSON.stringify(current) === JSON.stringify(next)) return;
current = next;
```
- **拓展**：可泛化到 v-model/表单联动等双向绑定，最好用单向数据流+唯一数据源根治。
- *来源：admin-workspace / ResizablePanels*

### 2. Quill 初始化时序陷阱
- **技能点**：掌握 watch immediate 与异步实例初始化时序，用缓存队列回放初始值。
- **坑点**：watch immediate 在 quill 未创建时触发，直接 return 丢失首次 modelValue，弹窗打开显示空白。
- **解决方案**：未就绪时把值存入 pendingModelValue，initQuill 后 nextTick 消费该缓存。
```text
let pending = null;
watch(() => props.modelValue, v => {
  quill ? quill.setContents(v) : (pending = v);
}, { immediate: true });
// initQuill:
await nextTick();
if (pending != null) setContent(pending);
```
- **拓展**：适用图表/Canvas/富文本等异步初始化需回放初始状态的组件。
- *来源：admin-workspace-new / QuillEditorNew*

### 3. Element Plus 按需引入 CSS
- **技能点**：识别按需注册组件样式缺失的根因，显式导入组件 CSS。
- **坑点**：unplugin-vue-components 只注册组件不自动补 CSS，splitter 等新组件样式缺失，布局退化为 block 堆叠。
- **解决方案**：显式 import 'element-plus/theme-chalk/el-splitter.css' 等，不用全量 CSS，逐个组件导入。
```text
import "element-plus/theme-chalk/el-splitter.css";
import "element-plus/theme-chalk/el-splitter-panel.css";
```
- **拓展**：按需加载的组件库常遇到，样式失效先查 CSS 是否真实加载。
- *来源：admin-workspace / MEMORY.md*

### 4. HTML 裸尖括号转义
- **技能点**：在 innerHTML 前识别并转义文本中的裸 < >，确保 HTML 结构闭合不被破坏。
- **坑点**：latex 含裸 < 如 0<b<1，innerHTML 解析器误判为标签开始，破坏闭合导致后续内容被合并。
- **解决方案**：用正则匹配 question-latex 标签内容，将其中 < > 转义为 &lt; &gt;，textContent 读取时自动解码。
```text
const escaped = html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi, (_, open, content, close) => open + content.replace(/</g, '&lt;').replace(/>/g, '&gt;') + close);
el.innerHTML = escaped;
```
- **拓展**：所有把含特殊字符文本嵌入 HTML 的场景都需转义；也可用 DOM 文本节点替代 innerHTML 拼接。
- *来源：admin-workspace-hr / v-katex*

### 5. Quill Clipboard matcher 补齐
- **技能点**：给自定义 Quill Clipboard 模块显式注册默认 matcher，避免吞掉图片和分隔线。
- **坑点**：TableClipboard 接管 clipboard 后只处理表格节点，没有 image/divider matcher，clipboard.convert 丢弃这些 embed。
- **解决方案**：在 registerClipboardMatchers 中为 img[data-type=ql-image]、divider.ql-divider 等添加 matcher，并构造与 Blot.value 对齐的 Delta。
```text
clipboard.addMatcher('img[data-type="ql-image"]', node => {
  return new Delta().insert({ image: { url: node.dataset.url, alt: node.dataset.alt } });
});
```
- **拓展**：新增自定义 embed blot 时也要显式加 matcher，不要假设会兜底继承。
- *来源：admin-workspace-new / MEMORY.md*

### 6. Table dataIndex 类型陷阱
- **技能点**：了解 rc-table dataIndex 类型过宽的事实，建立字段名人工核对机制。
- **坑点**：DataIndex<T> 的 SpecialString<T> 使任意字符串通过类型检查，列字段名写错不报错，渲染静默空白。
- **解决方案**：改行类型字段后逐列核对 dataIndex，同步 rowKey；可定义 Columns<T> 时用 keyof 辅助收窄。
```text
// TypeScript 无法从 rc-table 拿到 keyof 约束，需人工核对或自定义类型
const column: ColumnType<Row> = { dataIndex: 'typo_field', render: ... };
```
- **拓展**：用自定义类型或 lint 规则对 dataIndex 做 keyof 约束，避免运行时静默失败。
- *来源：admin-workspace-hr / antd Table 约定*

