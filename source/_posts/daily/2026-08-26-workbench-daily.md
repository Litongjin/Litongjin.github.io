---
title: "工作台日报 · 2026-08-26"
date: 2026-08-26 06:55:38
categories: [工作日记]
tags: ["日报", "AI", "大模型", "开源", "AI芯片"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-08-26

## 🔥 行业热点

- [OpenAI Jalapeño: Better than Nvidia Blackwell](https://newsletter.semianalysis.com/p/openai-jalapeno-better-than-nvidia) — *Hacker News*
  - 📌 **内容**：报道称OpenAI自研AI芯片Jalapeño性能优于Nvidia Blackwell，反映出大模型厂商加速自研算力。
  - 💡 **学习**：可关注AI芯片竞争对模型训练/推理成本的影响。
  - 🧭 **拓展**：后续可跟踪官方基准测试与量产进展。
- [How much of HN is AI?](https://blog.coredump.cx/p/how-much-of-hn-is-ai) — *Hacker News*
  - 📌 **内容**：统计Hacker News中AI相关帖子的占比，量化技术社区对AI话题的关注度。
  - 💡 **学习**：可学习用HN API和文本分类做社区趋势分析。
  - 🧭 **拓展**：尝试用Python抓取并复现统计。
- [Training AI to Paint with Code](https://surya.website/rling-qwen-to-paint-with-code) — *Hacker News*
  - 📌 **内容**：文章/项目介绍如何用代码引导AI生成绘画，将程序化生成与深度学习结合。
  - 💡 **学习**：可研究生成式模型与程序化笔触/构图的结合方法。
  - 🧭 **拓展**：用Stable Diffusion等模型尝试代码控制生成。
- [Show HN: I made a Raspberry with Qwen my local car AI](https://github.com/ThinkOffApp/CarWatch) — *Hacker News*
  - 📌 **内容**：作者用树莓派和Qwen模型打造车载本地AI助手，实现离线智能交互。
  - 💡 **学习**：学习在树莓派等低算力设备上部署大模型。
  - 🧭 **拓展**：可接入语音识别和车辆传感器扩展应用。
- [Show HN: TeXbrain, a LaTeX editor that runs pdfTeX in the browser via WASM](https://github.com/swimmingbrain/texbrain) — *Hacker News*
  - 📌 **内容**：TeXbrain是一个在浏览器中通过WASM运行pdfTeX的LaTeX编辑器，无需本地TeX环境。
  - 💡 **学习**：了解如何用WebAssembly将C/C++工具链移植到浏览器。
  - 🧭 **拓展**：可尝试用Emscripten编译其他排版工具。
- [Nitter project received cease and desist](https://github.com/zedeus/nitter/issues/1442) — *Hacker News*
  - 📌 **内容**：开源Twitter前端Nitter收到停止函，凸显第三方API合规风险。
  - 💡 **学习**：开发第三方客户端时需注意平台条款和知识产权。
  - 🧭 **拓展**：可研究开源项目应对法律挑战的策略。
- [What's new in Emacs 31.1](https://www.masteringemacs.org/article/whats-new-in-emacs-311) — *Hacker News*
  - 📌 **内容**：Emacs 31.1发布，包含编辑器与Lisp生态的多项更新。
  - 💡 **学习**：可通过更新日志学习Emacs Lisp的新特性和配置技巧。
  - 🧭 **拓展**：升级后测试个人配置兼容性。
- [Qwen 3.8-Flash-Next releasing tomorrow (125B a6B)](https://modelscope.cn/models/Qwen/Qwen3.8-Flash-Next) — *Hacker News*
  - 📌 **内容**：Qwen 3.8-Flash-Next将于明天发布，采用125B总参数/6B激活参数的MoE架构。
  - 💡 **学习**：理解MoE中总参数与激活参数对推理效率的影响。
  - 🧭 **拓展**：发布后可对比同级别模型的推理速度和效果。
- [Firefox 157 will include JPEG XL by default on all platforms](https://groups.google.com/a/mozilla.org/g/dev-platform/c/3YMV4MS34KA?pli=1) — *Hacker News*
  - 📌 **内容**：Firefox 157将在所有平台默认启用JPEG XL，推动下一代图片格式普及。
  - 💡 **学习**：了解JPEG XL的压缩率和Web应用价值。
  - 🧭 **拓展**：可在Firefox中实测JPEG XL图片加载性能。
- [SiFive's First Server Platform](https://chipsandcheese.com/p/sifives-first-server-platform) — *Hacker News*
  - 📌 **内容**：SiFive推出首款服务器平台，将RISC-V架构带入数据中心市场。
  - 💡 **学习**：关注RISC-V服务器生态和指令集演进。
  - 🧭 **拓展**：可对比RISC-V与ARM/x86在服务器场景的差异。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill Delta 与 HTML 同步
- **技能点**：掌握 vue-quill 编辑器内容同步时如何正确获取 HTML，避免 v-model 被 Delta JSON 污染。
- **坑点**：错用 getContents() 返回 Delta 对象，其 trim() 后是 Delta JSON 字符串，被当 HTML 解析导致匹配失败。
- **解决方案**：使用 getHTML() 或 quill.root.innerHTML 获取真实 HTML 字符串同步给 v-model。
```text
// 错误：getContents() 返回 Delta，trim() 后是 JSON 字符串
const delta = quillRef.value?.getContents().trim()
// 正确：getHTML() 返回真实 innerHTML
const html = quillRef.value?.getHTML?.() ?? quillRef.value?.root.innerHTML ?? ''
```
- **拓展**：涉及 Quill 内容提取、编辑器回显、服务端存储时都应使用 getHTML 而非 getContents。
- *来源：admin-workspace 2026-08-12*

### 2. HTML 标签解析引号感知
- **技能点**：编写 HTML 解析逻辑时，能识别属性值内特殊字符，避免误判标签边界。
- **坑点**：用 indexOf('>') 定位标签结束，遇到 data-formula="...(\omega>0..." 内原始 > 会截断标签，导致内容丢失。
- **解决方案**：扫描标签时若处于引号内则跳过，直到引号外的 > 才视为标签结束。
```text
function scanTagEnd(html, start) {
  let inQuote = ''
  for (let i = start; i < html.length; i++) {
    const ch = html[i]
    if (inQuote) {
      if (ch === inQuote) inQuote = ''
    } else if (ch === '"' || ch === "'") {
      inQuote = ch
    } else if (ch === '>') {
      return i
    }
  }
  return -1
}
```
- **拓展**：可沉淀为通用的 HTML 拍平/提取工具，处理任意属性内特殊字符。
- *来源：admin-workspace 2026-08-12*

### 3. innerHTML 前转义特殊字符
- **技能点**：在将含特殊字符的内容插入 innerHTML 前进行转义，防止浏览器 HTML5 解析器误判。
- **坑点**：latex 字符串含裸 <（如 `<b<`）直接 el.innerHTML = html，被解析器当成标签开始，破坏闭合结构，导致后续内容被吞并。
- **解决方案**：用正则将 `<question-latex>...</question-latex>` 内容中的 < > 转义为 &lt; &gt;，解析安全，textContent 读取时自动解码。
```text
html = html.replace(
  /(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi,
  (_, open, body, close) => open + body.replace(/</g, '&lt;').replace(/>/g, '&gt;') + close
)
el.innerHTML = html
```
- **拓展**：任何含公式/代码的富文本渲染都应先转义，或使用 textContent 等安全 API。
- *来源：admin-workspace MEMORY.md 2026-08-10*

### 4. 文本匹配容空白正则
- **技能点**：在文本匹配场景中构建容忍空白的正则，处理空格差异导致的失配。
- **坑点**：AI 生成的 original 与正文 latex 空格不一致（无空格 vs 带空格），严格字面匹配永远失败。
- **解决方案**：构建每个字符间允许任意空白的正则（\s*），严格匹配未命中时再容空白重试，并剔除首尾空白防吞边界。
```text
function buildWhitespaceTolerantRegex(str) {
  const escaped = str.split('').map(c => escapeRegExp(c)).join('\\s*')
  return new RegExp(escaped, 'g')
}
```
- **拓展**：可用于 OCR 文本比对、AI 纠错、搜索高亮等任何文本匹配场景。
- *来源：admin-workspace 2026-08-11*

### 5. HTML 实体整体解码
- **技能点**：处理 HTML 实体时，能整体解码并建立实体区间映射，避免逐字符解码失败。
- **坑点**：文本节点逐字符解码 &lt; 等实体失败（实体跨多字符），导致匹配 count=0。
- **解决方案**：累积文本节点后整体 decodeHtmlEntities，返回实体区间，字符 mapIndex 映射到实体起始偏移，片段边界延伸到实体末尾。
```text
function decodeHtmlEntities(str) {
  const div = document.createElement('div')
  div.innerHTML = str
  return div.textContent || ''
}
// 整体解码后再匹配，mapIndex 指向实体起始位置
```
- **拓展**：可推广到任何 HTML 到纯文本的映射场景，如富文本搜索、错别字定位。
- *来源：admin-workspace 2026-08-12*

### 6. 跨版本文件克隆保留结构
- **技能点**：在跨版本/跨目录复制文件时，能识别目标文件已有语义并保留，避免覆盖导致类型/结构损坏。
- **坑点**：直接用旧版 type.ts 覆盖新版，新版完整类型声明（QuillEditorProps 等）被旧版工具类替代，所有引用报错。
- **解决方案**：克隆前先查看目标文件已有导出；若目标文件承载语义，仅追加新代码，不替换底层结构；并校验 git diff 与 lint。
```text
// 不要：git checkout old -- path/type.ts
// 要：保留新版的类型声明，只追加旧版工具类
// git show HEAD:new/type.ts > new/type.ts
// git show HEAD:old/type.ts | grep 'ToolBtn\|btnStyle' >> new/type.ts
```
- **拓展**：可用于任何代码迁移、多版本共存、依赖升级场景。
- *来源：admin-workspace-new 2026-08-12*

### 7. Vue watch 双向同步守卫
- **技能点**：实现 Vue 组件状态与数组的 watch 双向同步时，能识别并避免无限递归。
- **坑点**：状态 A → 数组 → 状态 B → 状态 A 的 watch 链，即使值未变也写入新引用，触发 deep watch 循环直到 Maximum recursive updates。
- **解决方案**：每一步写入前先比较新旧值，无变化直接 return；引用类型对比内容而非引用。
```text
watch(localPanelSizes, (val) => {
  const next = deriveFromSizes(val)
  if (isFilePreviewFolded.value !== next.left) {
    isFilePreviewFolded.value = next.left
  }
  // 值未变化时不回写，打破循环
})
```
- **拓展**：适用于任何多状态互相 watch 同步的场景，如面板尺寸、折叠状态、配置同步。
- *来源：admin-workspace 2026-08-11*

