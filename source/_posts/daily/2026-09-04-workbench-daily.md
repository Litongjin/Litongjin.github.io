---
title: "工作台日报 · 2026-09-04"
date: 2026-09-04 07:01:36
categories: [工作日记]
tags: ["日报", "AI", "AI服务", "大模型", "AGI"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-04

## 🔥 行业热点

- [Xanadu was waiting for agents](https://zed.dev/blog/agentic-xanadu) — *Hacker News*
  - 📌 **内容**：回顾 Xanadu 超文本系统，并讨论其设计如何预示了今天的 Agent 概念。
  - 💡 **学习**：从早期系统设计理解 Agent 交互与协议需求。
  - 🧭 **拓展**：可对比 Xanadu 与 Web/Agent 技术路线。
- [Qwen 3.8 27B available on Cerebras at 1500 tokens/s](https://inference-docs.cerebras.ai/models/overview) — *Hacker News*
  - 📌 **内容**：Qwen 3.8 27B 模型上线 Cerebras 平台，推理吞吐达到 1500 tokens/s。
  - 💡 **学习**：了解专用 AI 硬件如何提升大模型推理速度与成本效率。
  - 🧭 **拓展**：可通过 Cerebras API 实测并对比其他推理平台。
- [The browser's main thread is expensive](https://kciter.so/posts/the-expensive-main-thread/en/) — *Hacker News*
  - 📌 **内容**：文章强调浏览器主线程开销对前端性能的影响，长任务会阻塞交互。
  - 💡 **学习**：学会识别主线程长任务，并用 Web Worker 或异步渲染优化。
  - 🧭 **拓展**：可用 DevTools Performance 面板测量自己的页面。
- [Ask HN: Why were OpenAI, Claude, and Grok simultaneously down?](https://news.ycombinator.com/item?id=49551096) — *Hacker News*
  - 📌 **内容**：HN 用户询问 OpenAI、Claude、Grok 同时宕机的原因，可能指向共享基础设施或外部依赖。
  - 💡 **学习**：多服务同时故障提醒关注依赖耦合与容灾设计。
  - 🧭 **拓展**：可观察各服务状态页及事后分析报告。
- [Go grandmaster Shin defeats AI KataGo with a two-stone handicap](https://www.kedglobal.com/artificial-intelligence/newsView/ked202607210007) — *Hacker News*
  - 📌 **内容**：围棋高手 Shin 在让两子条件下击败 AI KataGo，展示 AI 围棋策略上的可攻击性。
  - 💡 **学习**：可研究棋类 AI 在非对称规则下的鲁棒性。
  - 🧭 **拓展**：可结合对抗性输入思路分析模型盲区。
- [OpenAI's GPT-6 Astra on ARC-AGI-3](https://arcprize.org/blog/astra) — *Hacker News*
  - 📌 **内容**：OpenAI 的 GPT-6 Astra 在 ARC-AGI-3 基准上的表现引发关注，该基准考验抽象推理。
  - 💡 **学习**：可通过 ARC 类任务理解大模型在推理泛化上的边界。
  - 🧭 **拓展**：可查阅基准细节并尝试复现评测。
- [Audacity 4.0](https://github.com/audacity/audacity/releases/tag/Audacity-4.0.0) — *Hacker News*
  - 📌 **内容**：开源音频编辑器 Audacity 发布 4.0 大版本，带来功能与界面更新。
  - 💡 **学习**：关注开源桌面软件在跨平台与音频处理上的演进。
  - 🧭 **拓展**：可下载试用并查看官方发布说明。
- [Pre-Release of Polars 2.0](https://pola.rs/posts/announcing-polars-2/) — *Hacker News*
  - 📌 **内容**：Polars 2.0 预发布，DataFrame 库在性能、API 上预计有重要变化。
  - 💡 **学习**：关注 Rust/Python 数据处理库的接口演进和查询优化。
  - 🧭 **拓展**：可在现有 ETL 流程中做兼容性测试。
- [Nvidia to acquire Hugging Face](https://www.cnbc.com/2026/09/03/nvidia-agrees-to-buy-hugging-face-for-almost-13-billion-ai-expansion.html) — *Hacker News*
  - 📌 **内容**：Nvidia 将收购 Hugging Face，AI 基础设施与开源模型平台的整合进一步加速。
  - 💡 **学习**：思考算力厂商与模型平台整合对开发者生态的影响。
  - 🧭 **拓展**：关注后续开源模型托管与推理服务变化。
- [Google Antigravity TOS: 3rd party usage can get Google account suspended](https://twitter.com/GergelyOrosz/status/2095453567955968398) — *Hacker News*
  - 📌 **内容**：Google Antigravity 的服务条款显示，第三方违规使用可能导致 Google 账号被封。
  - 💡 **学习**：使用 AI 服务前需仔细阅读 TOS，避免账号关联风险。
  - 🧭 **拓展**：可对比其他 AI 平台的违规处罚条款。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步递归更新
- **技能点**：掌握 Vue watch 双向同步时防止无限递归的守卫写法，避免状态互相触发。
- **坑点**：两个 watch 互相触发（状态A→数组→状态B→状态A），即使值未变也因新数组引用+deep watch 导致 Maximum recursive updates。
- **解决方案**：每次写回前比较目标值是否变化，无变化直接 return；不要在 watch 中无条件赋值新引用。
```text
watch([flagA, flagB], () => {
  const next = [...];
  if (next.every((v,i)=>v===localSizes[i])) return;
  localSizes.value = next;
});
watch(localSizes, (val) => {
  if (val[0] !== flagA.value) flagA.value = val[0];
  // ...
}, { deep: true });
```
- **拓展**：可提炼为通用模式：跨 watch 同步状态时先比较再赋值；或改用单一数据源避免双向同步。
- *来源：admin-workspace 2026-08-11*

### 2. Quill getContents 返回 Delta 而非 HTML
- **技能点**：明确 vue-quill 的 getContents() 返回 Delta，getHTML()/root.innerHTML 才返回 HTML。
- **坑点**：watch 同步 v-model 时误用 getContents()，Delta JSON 串被当 HTML 解析，导致内容被转义、匹配失败。
- **解决方案**：使用 quillRef.value?.getHTML?.() ?? '' 取真实 innerHTML，或直接读 quill.root.innerHTML。
```text
quillRef.value?.getContents().trim() // 错
quillRef.value?.getHTML?.() ?? '' // 对
```
- **拓展**：任何富文本编辑器同步 v-model 都应取序列化后的 HTML，而不是内部数据结构。

### 3. HTML 属性内特殊字符需实体化
- **技能点**：理解 HTML 解析规则，将 latex 等含 < > 的内容存入属性时必须实体化，解析时需引号感知扫描标签边界。
- **坑点**：data-formula="...\\omega>0..." 中原始 > 使 html.indexOf('>') 提前截断标签，导致内容丢失、匹配失败。
- **解决方案**：标签结束定位改为引号感知扫描：< 后进入引号则跳过，直到引号外遇到真正 >；存储侧统一 encodeHtmlEntities。
```text
let i = html.indexOf('<', pos);
while (i !== -1) {
  const quote = html[i+1] === '"' ? '"' : html[i+1] === "'" ? "'" : null;
  // 引号内跳过，直到引号外遇到 >
}
```
- **拓展**：可沉淀为通用的 HTML 拍平解析工具，处理属性值含特殊字符的场景。

### 4. AI 文本匹配需容忍空白差异
- **技能点**：构建容错匹配正则，处理 AI 生成内容与正文在空格上的不一致。
- **坑点**：original 无空格、正文 latex 带空格时，严格字面匹配永远失配，导致高亮/替换漏掉。
- **解决方案**：每个字符间允许 \s*，纯空白返回 null 防死循环；严格匹配优先，未命中再容空白重试，并剔除首尾空白。
```text
const buildWhitespaceTolerantRegex = (s) => s.trim() ? new RegExp(s.split('').map(escapeRegExp).join('\\s*'), 'g') : null;
```
- **拓展**：可扩展到容忍其他噪声（如换行、全半角）的模糊匹配工具。

### 5. element-plus 按需注册需显式引 CSS
- **技能点**：理解 unplugin-vue-components 按需注册组件与加载 CSS 是两回事，新组件要显式 import 其样式。
- **坑点**：el-splitter/panel 等较新组件在按需注册时 CSS 未被自动引入，布局退化为普通 block 导致堆叠。
- **解决方案**：显式 import 'element-plus/theme-chalk/el-splitter.css' 等；不要依赖全量 index.css。
```text
import 'element-plus/theme-chalk/el-splitter.css';
import 'element-plus/theme-chalk/el-splitter-panel.css';
```
- **拓展**：可建立项目内 element-plus 按需引入清单，新增组件时同步补 CSS。
- *来源：admin-workspace MEMORY.md*

### 6. antd Table sticky 必须配 scroll.x
- **技能点**：掌握 antd Table 粘性表头与横向滚动共存的约束。
- **坑点**：启用 sticky 未设 scroll.x，大容器下表头与 body 右侧列错位。
- **解决方案**：sticky 与 scroll={{ x: 'max-content' }} 同时声明。
```text
<Table sticky scroll={{ x: 'max-content' }} ... />
```
- **拓展**：可沉淀为 antd 表格配置检查清单，避免同类错位。
- *来源：admin-workspace-hr MEMORY.md*

