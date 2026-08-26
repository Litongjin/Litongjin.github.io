---
title: "工作台日报 · 2026-08-27"
date: 2026-08-27 06:55:58
categories: [工作日记]
tags: ["日报", "大模型", "AI", "AI Agent", "AI工具"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-08-27

## 🔥 行业热点

- [Serve Markdown to AI Agents with Accept Headers](https://acceptmarkdown.com/) — *Hacker News*
  - 📌 **内容**：Discusses using HTTP Accept headers to serve Markdown content optimized for AI agents, enabling better content negotiation.
  - 💡 **学习**：Learn how to leverage Accept headers to deliver structured content to AI crawlers and agents.
  - 🧭 **拓展**：Experiment by adding an Accept header handler to your own API.
- [Show HN: Devx – Autonomous AI coding agent built for Android Termux and desktop](https://github.com/apvcode/Termux-Dev) — *Hacker News*
  - 📌 **内容**：Devx is an autonomous AI coding agent that runs on Android Termux and desktop environments.
  - 💡 **学习**：Explore how AI coding agents can operate in terminal and mobile environments.
  - 🧭 **拓展**：Try running Devx in Termux to see its capabilities.
- [Why AI Agents Need Persistent Browser Identities](https://github.com/Radek-B3/browser3/blob/main/WHY_AI_AGENTS_NEED_PERSISTENT_BROWSER_IDENTITIES.md) — *Hacker News*
  - 📌 **内容**：Argues that AI agents require persistent browser identities to maintain state and context across sessions.
  - 💡 **学习**：Understand the role of session persistence in agent-based automation.
  - 🧭 **拓展**：Investigate browser fingerprinting and profile management for agents.
- [Tailcat – Like netcat, but over Tailscale’s data plane](https://github.com/tailscale/tailcat) — *Hacker News*
  - 📌 **内容**：Tailcat is a network tool similar to netcat but built on Tailscale's encrypted data plane.
  - 💡 **学习**：Learn how Tailscale enables secure peer-to-peer networking for simple utilities.
  - 🧭 **拓展**：Use Tailcat to test connectivity between tailnet devices.
- [PageRank explained](https://praveshkoirala.com/2026/08/26/you-could-have-invented-pagerank/) — *Hacker News*
  - 📌 **内容**：A clear explanation of the PageRank algorithm that powers search engines.
  - 💡 **学习**：Understand the math behind link-based ranking and its iterations.
  - 🧭 **拓展**：Implement a small PageRank in Python to solidify the concept.
- [Show HN: We built the smallest dual-band aircraft tracker](https://pantsforbirds.com/the-worlds-smallest-dual-band-ads-b-receiver-module/) — *Hacker News*
  - 📌 **内容**：A team showcases a compact dual-band tracker for aircraft, combining hardware and software.
  - 💡 **学习**：Explore embedded systems design for RF signal processing.
  - 🧭 **拓展**：Check the project’s schematic and firmware for learning.
- [WebMCP Challenge – OpenAI](https://openai.com/webmcp-challenge/) — *Hacker News*
- [AWS Acquires DuckLabs](https://ducklabs.com/news/2026/08/26/ducklabs-to-join-aws) — *Hacker News*
  - 📌 **内容**：AWS has acquired DuckLabs, signaling expansion in a new technical domain.
  - 💡 **学习**：Consider how cloud providers integrate acquisitions into their platform.
  - 🧭 **拓展**：Watch for DuckLabs products being rolled into AWS services.
- [GLM-5.3-Flash](https://z.ai/blog/glm-5.3-flash) — *Hacker News*
  - 📌 **内容**：Release of GLM-5.3-Flash, a fast version of the GLM large language model.
  - 💡 **学习**：Review the model's architecture, speed, and benchmark results.
  - 🧭 **拓展**：Test the model through its API for latency-sensitive applications.
- [RAG Is Simpler Than You Think](https://www.lighthousenewsletter.com/p/rag-is-simpler-than-you-think) — *Hacker News*
  - 📌 **内容**：Explains that Retrieval-Augmented Generation (RAG) is conceptually simpler than many believe.
  - 💡 **学习**：Grasp the basic components: retriever, index, and generation model.
  - 🧭 **拓展**：Build a minimal RAG pipeline with a vector database.

## 🚀 技能提升点（工作总结汇总）

### 1. HTML 拍平匹配与偏移回写
- **技能点**：将富文本 HTML 拍平为纯文本并建立偏移映射，在纯文本上匹配后回写高亮/替换到原 HTML 片段；掌握引号感知的标签边界解析。
- **坑点**：旧实现把标签替换为空格且不折叠，标签边界空格数不一致、跨标签字面匹配失败；indexOf('>') 定位标签结束被 data-formula 属性值内原始 > 截断；HTML 实体逐字符解码也失败。
- **解决方案**：标签→空串（不产生空格），formula 整体原子映射到开标签位置；文本节点累积后整体 decodeHtmlEntities 并记录 entityRanges；标签结束用引号感知扫描（引号内跳过，引号外 > 才算结束）。
```text
// 引号感知定位标签结束：data-formula 属性内原始 > 不截断标签
let j = i + 1, quote: string | null = null;
for (; j < html.length; j++) {
  const ch = html[j];
  if (quote) { if (ch === quote) quote = null; continue; }
  if (ch === '"' || ch === "'") { quote = ch; continue; }
  if (ch === '>') break;
}
// 标签 → 空串（不产生空格）；formula 整体原子映射到开标签位置
```
- **拓展**：可沉淀为通用富文本标注库，推广到拼写检查、敏感词高亮、OCR 校对等场景。
- *来源：admin-workspace 2026-08-12*

### 2. 容空白匹配
- **技能点**：构建容空白正则（空格位置允许零或多个空格）处理 AI 生成文本与正文的空白不一致，掌握严格优先、容错兜底的匹配策略。
- **坑点**：escapeRegExp(original) 严格字面匹配，original 无空格、正文 latex 带空格（如 \frac {\pi}{2} vs \frac{\pi}{2}）时永远失配。
- **解决方案**：buildWhitespaceTolerantRegex 把字面空格替换为 [ ]* 实现容错；纯空白字符串返回 null 防零宽无限匹配；严格匹配未命中才容空白重试，collectRanges 剔除首尾空白防范围越界。
```text
function buildWhitespaceTolerantRegex(str: string) {
  if (!str.trim()) return null;
  return new RegExp(escapeRegExp(str).replace(/ /g, '[ ]*'));
}
```
- **拓展**：可推广到搜索、去重、校对等机器文本 vs 库中文本比对场景，也可改为统一空白归一化后匹配。
- *来源：admin-workspace 2026-08-12*

### 3. Quill getContents 与 getHTML 区分
- **技能点**：明确 vue-quill 的 getContents() 返回 Delta 对象、getHTML() 才返回 HTML 字符串，避免把 Delta JSON 当 HTML 处理。
- **坑点**：quillRef.value?.getContents().trim() 得到 Delta JSON 字符串后被 DOMParser 当 HTML 解析，< 被转义为 &lt;，后续按标签/属性匹配的逻辑全部失效，v-model 被污染为 Delta JSON 文本。
- **解决方案**：统一用 quillRef.value?.getHTML?.() ?? ''（或 quill.root.innerHTML）取真实 Quill innerHTML 再交给 HTML 侧处理。
```text
// bad：getContents() 返回 Delta，序列化成 JSON 字符串被当 HTML
const html = quillRef.value?.getContents().trim();
// good：getHTML() 才返回真实 HTML
const html = quillRef.value?.getHTML?.() ?? quillRef.value?.root?.innerHTML ?? '';
```
- **拓展**：凡 Quill 内容同步到外部字符串或做文本分析都必须走 getHTML，不要用 getContents 的 Delta 序列化当 HTML。
- *来源：admin-workspace 2026-08-12*

### 4. Quill toolbar 配置展平
- **技能点**：理解 Quill toolbar controls 是扁平对象数组，配置映射返回多按钮数组时必须 flatMap 展平。
- **坑点**：order.map(...) 中混入数组项（如 [{list:'ordered'},{list:'bullet'}]），Quill addControls 把数组当对象，Object.keys(control)[0]==='0' 当 format 名 → 生成 ql-0 空按钮、value="[object Object]"。
- **解决方案**：order.flatMap(...) 展平为一维 controls，每个 {list:'ordered'} 独立成 control，Quill 正常识别。
```text
const controls = toolbarOrder.flatMap((name) => {
  const cfg = getDefaultButtonConfig(name);
  return Array.isArray(cfg) ? cfg : [cfg];
});
```
- **拓展**：同理适用于任何配置映射成控件列表的框架，多值映射时留意目标 API 是否接受嵌套结构。
- *来源：admin-workspace-new 2026-08-12*

### 5. Vue watch 双向同步防循环
- **技能点**：掌握 Vue watch 双向同步（状态A→数组→状态B→状态A）的循环防护：每一步回写前先做值比较，无变化不写。
- **坑点**：折叠标志 → 写 localPanelSizes（新数组引用）→ deep watch 触发 → 写回折叠标志（值未变也写）→ 再触发 watch → 无限递归直至 Maximum recursive updates exceeded。
- **解决方案**：同步函数先计算 next 并逐项比对，相等直接 return；deep watch 内对每个目标标志先比较当前值 !== 推导值再回写。
```text
const next = [leftWidth, 'auto', rightWidth];
if (next.every((v, i) => v === localPanelSizes.value[i])) return; // 无变化不写
localPanelSizes.value = next;
// deep watch 内：值比较守卫
if (isFilePreviewFolded.value !== derived.left) isFilePreviewFolded.value = derived.left;
if (isAiCheckContentFolded.value !== derived.right) isAiCheckContentFolded.value = derived.right;
```
- **拓展**：任何多状态互相镜像（面板尺寸/折叠/持久化）都应采用单向数据流 + 值比较守卫，而不是裸 watch 互写。
- *来源：admin-workspace 2026-08-11*

### 6. 按需注册组件需显式引 CSS
- **技能点**：识别 unplugin-vue-components + ElementPlusResolver 按需注册组件时不自动补齐组件 CSS 的坑，排查布局不生效先确认 CSS 是否加载。
- **坑点**：el-splitter 等较新组件 CSS 未被 resolver 引入，.el-splitter 退化为普通 block 容器，三栏堆叠成竖排；反复调 flex-direction 治标不治本。
- **解决方案**：在组件文件显式 import element-plus/theme-chalk/el-splitter.css（及 el-splitter-panel.css），不依赖全量 dist CSS。
```text
import "element-plus/theme-chalk/el-splitter.css";
import "element-plus/theme-chalk/el-splitter-panel.css";
```
- **拓展**：排查 UI 组件布局不生效时，先确认对应组件 CSS 是否真的被加载，再怀疑样式覆盖或布局属性。
- *来源：admin-workspace MEMORY.md*

