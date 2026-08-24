---
title: "工作台日报 · 2026-08-25"
date: 2026-08-25 06:56:15
categories: [工作日记]
tags: ["日报", "大模型", "开源", "AI", "AI工具"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-08-25

## 🔥 行业热点

- [MS Paint and Photos inivisibly watermark even locally generated output with GUID](https://xusheng.dev/posts/reversing/mspaint_invisible_watermark/main/) — *Hacker News*
  - 📌 **内容**：微软画图与照片应用即使在本地生成的内容中也会嵌入GUID水印，引发用户对隐私和内容溯源的新关注。
  - 💡 **学习**：开发图像处理工具时应了解元数据与隐式水印机制，避免无意间泄露用户身份信息。
  - 🧭 **拓展**：可用exiftool或二进制编辑器检查本地图片是否包含GUID。
- [Coding expertise is going to collapse from AI reliance](https://larsfaye.com/articles/ai-coding-will-prevent-expertise) — *Hacker News*
  - 📌 **内容**：文章担忧过度依赖AI编程工具会导致开发者深层编码能力退化，行业整体专业水平下降。
  - 💡 **学习**：在AI辅助下保持代码审查、调试与底层原理学习，是维持专业能力的关键。
  - 🧭 **拓展**：可尝试无AI工具完成复杂模块，对比能力变化。
- [I built a low-latency AI companion that plays Skyrim with me](https://pantel.is/projects/ai-gaming-companion/) — *Hacker News*
  - 📌 **内容**：作者开发了一个低延迟AI同伴，能实时陪玩《上古卷轴5》，展示AI在游戏交互中的新可能。
  - 💡 **学习**：学习如何将大模型与游戏状态流结合，构建低延迟实时Agent。
  - 🧭 **拓展**：可尝试用类似方法为其他游戏添加AI同伴。
- [IPFS Maintainers Winding Down](https://ipshipyard.com/blog/2026-the-end-of-ipfs-at-shipyard/) — *Hacker News*
  - 📌 **内容**：IPFS维护者宣布逐步结束维护工作，去中心化存储生态面临重要转折。
  - 💡 **学习**：关注分布式存储项目的治理与可持续性，评估技术选型风险。
  - 🧭 **拓展**：可研究Filecoin等其他去中心化存储方案的现状。
- [OpenAI: GPT 5.6 Sol price reduction (until at least Nov 21)](https://developers.openai.com/api/docs/pricing) — *Hacker News*
  - 📌 **内容**：OpenAI宣布GPT-5.6价格下调，并至少持续到11月21日。
  - 💡 **学习**：持续关注大模型API定价变化，有助于优化应用成本结构。
  - 🧭 **拓展**：可对比新旧价格并测算现有应用的成本影响。
- [Show HN: Kern – container and resource runtime in a 1.5 MB binary, no daemon](https://github.com/getkern/kern) — *Hacker News*
  - 📌 **内容**：Kern以1.5MB单二进制提供容器与资源运行时，无需守护进程，追求极简轻量。
  - 💡 **学习**：学习容器运行时的核心机制与静态编译优化。
  - 🧭 **拓展**：可对比Kern与runc、containerd的架构差异。
- [Qwen 3.6 is now much easier to run locally on your Mac, thanks to JetBrains](https://www.neowin.net/news/qwen-36-is-now-much-easier-to-run-locally-on-your-mac-thanks-to-jetbrains/) — *Hacker News*
  - 📌 **内容**：JetBrains的优化让Qwen 3.6在Mac上本地运行更加简单，降低了大模型本地部署门槛。
  - 💡 **学习**：可学习大模型在Apple Silicon上的量化与推理加速技巧。
  - 🧭 **拓展**：尝试在Mac上部署Qwen 3.6并测试推理速度。
- [Xiaomi: New CPU matches Apple cores single threaded, much faster multithreaded](https://twitter.com/lemire/status/2091894299289874926) — *Hacker News*
  - 📌 **内容**：小米新款CPU单核性能对标Apple核心，多核性能明显更强，引发芯片领域关注。
  - 💡 **学习**：关注CPU微架构设计与多核调度优化。
  - 🧭 **拓展**：可查看具体基准测试与能效数据。
- [Executable Is a SQLite Database](https://fzakaria.com/2026/08/23/your-executable-is-a-sqlite-database) — *Hacker News*
  - 📌 **内容**：一种巧妙的技术将可执行文件同时作为SQLite数据库使用，实现程序与数据的统一。
  - 💡 **学习**：学习ELF/可执行文件格式、SQLite VFS与自定义段的技术。
  - 🧭 **拓展**：可尝试用此方法构建自包含数据应用。
- [SeL4 security proofs now complete on AArch64](https://proofcraft.systems/news-2026/#2026-08-21) — *Hacker News*
  - 📌 **内容**：seL4微内核在AArch64架构上的安全证明正式完成，为高安全系统提供更强保障。
  - 💡 **学习**：学习形式化验证在操作系统安全中的应用。
  - 🧭 **拓展**：可阅读seL4的证明方法与验证工具链。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill Delta 与 HTML 取值区分
- **技能点**：掌握 Quill 内容取值 API 差异，能避免将 Delta 对象误当 HTML 字符串处理。
- **坑点**：vue-quill 的 getContents() 返回 Delta 对象，trim() 后得到 JSON 字符串，被 DOMParser 当 HTML 解析后污染 v-model，导致后续匹配/替换逻辑全部失效。
- **解决方案**：同步 modelValue 时改用 quillRef.value?.getHTML?.() ?? "" 取真实 innerHTML。
```text
const html = quillRef.value?.getHTML?.() ?? '';
```
- **拓展**：可沉淀为 Quill 封装规范：对外输出统一走 getHTML，避免业务侧误用 Delta。
- *来源：admin-workspace 2026-08-12.md*

### 2. 克隆文件前检查目标结构
- **技能点**：跨版本/跨目录复用文件时，先分析目标文件既有导出与语义，避免破坏底层类型定义。
- **坑点**：直接用旧版 type.ts 覆盖新版完整类型声明，导致所有引用 QuillEditorProps 的位置类型解析失败。
- **解决方案**：保留新版原始类型声明，再追加旧版工具函数；用 git show 对比源与目标内容，确认是“新增”而非“替换”。
```text
git show HEAD:src/component/tool/quillEditorNew/type.ts
```
- **拓展**：任何文件覆盖前先 diff，判断目标文件是否承载独立语义，避免同名导出被误覆盖。
- *来源：admin-workspace 2026-08-12.md*

### 3. HTML 中 LaTeX 裸尖括号实体化
- **技能点**：理解浏览器 HTML 解析对 < > 的敏感性，掌握将 LaTeX 内容安全嵌入 HTML 属性/文本的方法。
- **坑点**：把含 < > 的原始 LaTeX 直接塞进 data-formula 或 innerHTML，导致标签提前截断、内容被吞或高亮失配。
- **解决方案**：写入 HTML 前将 < > 转义为 &lt; &gt;；读取时用 textContent 自动解码回原 LaTeX。
```text
el.innerHTML = html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi, (_, s, body, e) => `${s}${body.replace(/</g,'&lt;').replace(/>/g,'&gt;')}${e}`);
```
- **拓展**：可封装统一的 latex-to-html 工具，集中处理转义与还原，避免各解析侧分别兜底。
- *来源：admin-workspace MEMORY.md 2026-08-10.md*

### 4. Vue watch 双向同步防递归
- **技能点**：设计跨状态同步时使用值比较守卫，避免 watch 互相触发无限递归。
- **坑点**：A → 数组 → B → A 的双向 watch 中，即使值未变也写入新数组/标志，导致 Maximum recursive updates exceeded。
- **解决方案**：每次推导后先与当前值比较，无变化则直接 return；deep watch 内回写前同样判断。
```text
watch(localPanelSizes, (v) => {
  const next = deriveFlag(v);
  if (next !== currentFlag.value) currentFlag.value = next;
});
```
- **拓展**：可抽象 useSyncedRefs 或约定 watch 回写必须带 guard，防止引用变化触发循环。
- *来源：admin-workspace 2026-08-11.md*

### 5. Quill clipboard matcher 显式注册
- **技能点**：自定义 Clipboard 模块时，知道默认 matcher 不会自动继承，需手动补齐关键节点。
- **坑点**：注册 TableClipboard 接管 clipboard 后，<img data-type="ql-image"> 与 .ql-divider 被丢弃，富文本图片/分割线丢失。
- **解决方案**：在 registerClipboardMatchers 中显式 addMatcher，返回与对应 Blot.value 结构对齐的 Delta。
```text
clipboard.addMatcher('img[data-type="ql-image"]', node => new Delta().insert({ image: extractImageAttrs(node) }));
```
- **拓展**：新增自定义 embed blot 时沿用同一模式，不依赖第三方模块兜底默认行为。
- *来源：admin-workspace-new MEMORY.md*

### 6. HTML 纯文本拍平与偏移映射
- **技能点**：设计 HTML 与纯文本双模匹配时，用 mapIndex 维护字符级映射，支持标签/公式原子处理。
- **坑点**：用正则把标签替换成空格再匹配，标签边界空格数不一致，且跨标签 original 字面匹配失败。
- **解决方案**：标签→空串并记录偏移映射，匹配在纯文本上做，回写时按 mapIndex 映射回 HTML，公式整段原子包裹。
```text
const { plain, mapIndex } = flattenToPlain(html);
const [pStart, pEnd] = matchOnPlain(plain, original);
const htmlRange = mapToHtmlRange(mapIndex, pStart, pEnd);
```
- **拓展**：可复用为富文本高亮/替换/还原的通用工具，注意实体与公式子串粒度。
- *来源：admin-workspace 2026-08-12.md*

