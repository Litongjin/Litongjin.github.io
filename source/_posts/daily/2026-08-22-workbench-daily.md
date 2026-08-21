---
title: "工作台日报 · 2026-08-22"
date: 2026-08-22 07:00:49
categories: [工作日记]
tags: ["日报", "AI", "AI Agent", "AI工具", "AI推理"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-08-22

## 🔥 行业热点

- [AI boosted homework scores, then exam scores dropped: study](https://www.economist.com/graphic-detail/2026/08/18/does-ai-stop-children-from-learning) — *Hacker News*
  - 📌 **内容**：一项研究显示，使用AI辅助完成作业的学生在作业成绩上有所提升，但考试分数出现下滑，暗示AI辅助可能削弱了知识内化。
  - 💡 **学习**：开发者应关注AI工具对用户长期学习效果的影响，思考如何设计既提升效率又不损害能力的辅助系统。
  - 🧭 **拓展**：可结合教育科技产品设计A/B测试，对比使用AI辅助与自主完成的学习效果。
- [Building an (almost) fully self-hosted, sandboxed, agentic software factory](https://blog.jakesaunders.dev/building-an-almost-fully-self-hosted-sandboxed-agentic-software-factory/) — *Hacker News*
  - 📌 **内容**：文章介绍如何搭建一个近乎完全自托管、沙箱隔离且具备自主代理能力的软件开发工厂，强调可控与安全。
  - 💡 **学习**：学习自托管开发环境与沙箱隔离技术，了解如何为AI编码代理提供安全执行空间。
  - 🧭 **拓展**：可尝试基于本地容器或虚拟机搭建类似的自托管CI/CD与Agent运行环境。
- [Show HN: Proliferate- open-source, self-hostable Codex for any coding agent](https://github.com/proliferate-ai/proliferate) — *Hacker News*
  - 📌 **内容**：Proliferate 是一个开源自托管项目，旨在为任意编码代理提供类似Codex的能力。
  - 💡 **学习**：了解如何构建可插拔的编码代理后端，实现模型无关的代码生成接口。
  - 🧭 **拓展**：阅读源码或部署体验，对比不同模型在代码生成任务上的表现。
- [Kagi added a setting for removing paywalled links from search results](https://kagi.com/changelog#11296) — *Hacker News*
  - 📌 **内容**：Kagi 搜索引擎新增设置项，允许用户从搜索结果中过滤掉有付费墙的链接。
  - 💡 **学习**：关注搜索引擎产品功能设计，理解用户对内容可访问性的需求。
  - 🧭 **拓展**：可研究付费墙检测算法，并思考如何在小众搜索引擎中实现类似过滤。
- [DeepSeek-v4-flash-vision-exp](https://api-docs.deepseek.com/guides/vision/) — *Hacker News*
  - 📌 **内容**：DeepSeek 新模型版本曝光，定位为视觉能力实验型模型，名称暗示具备多模态理解能力。
  - 💡 **学习**：可关注多模态大模型的最新进展，了解视觉语言模型的实验性设计。
  - 🧭 **拓展**：尝试在本地或API上测试该模型的视觉推理能力，与既有模型做对比。
- [Small, native web tricks worth remembering](https://htmlcat.net/) — *Hacker News*
  - 📌 **内容**：文章汇总了一些值得记住的小型原生Web开发技巧，强调不依赖框架的简洁实现。
  - 💡 **学习**：复习原生HTML/CSS/JavaScript的实用技巧，提升对浏览器平台能力的掌握。
  - 🧭 **拓展**：在实际项目中刻意使用原生API替代框架，检验性能与可维护性。
- [Claudette: Make Claude stop talking like a BuzzFeed article](https://github.com/adnanakil/nobuzz/blob/main/README.md) — *Hacker News*
  - 📌 **内容**：Claudette 是一个用于调整Claude输出风格的工具或提示方法，让回复不再像BuzzFeed式营销文案。
  - 💡 **学习**：学习如何通过提示工程或后处理来约束大模型的语气与风格，满足产品调性。
  - 🧭 **拓展**：可尝试编写系统提示词或微调脚本来控制模型的口语化程度。
- [The coolest anti-surveillance tools at Defcon [video]](https://www.youtube.com/watch?v=-2uAsJ5EPAw) — *Hacker News*
  - 📌 **内容**：视频展示了Defcon大会上亮相的反监控工具，涵盖对抗摄像头、追踪和身份识别的技术。
  - 💡 **学习**：了解反监控与隐私保护技术，包括干扰信号、匿名通信等方法。
  - 🧭 **拓展**：可研究这些工具背后的原理，如红外干扰、人脸识别规避等，并思考合法使用边界。
- [I ran Photoshop on a £0.60 computer chip](https://pointinthecloud.com/2026-08-19-144600.html) — *Hacker News*
  - 📌 **内容**：作者成功在成本仅60便士的廉价芯片上运行了Photoshop，展示了低功耗硬件的能力。
  - 💡 **学习**：思考如何在资源受限设备上运行桌面级应用，涉及系统优化与软件适配。
  - 🧭 **拓展**：可尝试在树莓派或类似设备上复现，记录性能瓶颈与调优过程。
- [How we made a text-to-speech model respond in sub-50 ms](https://nari-labs.com/blog/qwen3-tts-speed-cost-frontier/) — *Hacker News*
  - 📌 **内容**：文章介绍了如何将文本转语音模型的响应时间压缩到50毫秒以内，实现接近实时的语音反馈。
  - 💡 **学习**：学习低延迟推理的优化手段，如流式生成、模型剪枝、缓存与推理引擎调优。
  - 🧭 **拓展**：可基于开源TTS模型做延迟基准测试，对比不同优化策略的效果。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill 内容同步用 getHTML
- **技能点**：掌握 vue-quill 编辑器中同步 v-model 必须用 getHTML() 获取真实 HTML，而非 getContents() 返回的 Delta 对象。
- **坑点**：getContents() 返回 Delta，后续 trim() 是 Delta 方法，将 Delta JSON 字符串当 HTML 用 DOMParser 解析，导致 v-model 被污染为 Delta JSON 文本。
- **解决方案**：统一使用 quillRef.value?.getHTML?.() ?? '' 获取 Quill 的 innerHTML（含 data-formula 等真实标签）。
```text
// 错误：返回 Delta 对象，JSON 字符串污染 v-model
const delta = quillRef.value?.getContents().trim()
// 正确：返回 HTML 字符串
const html = quillRef.value?.getHTML?.() ?? ""
```
- **拓展**：可沉淀为 Quill 封装组件的标准内容获取规范，所有同步 v-model 的地方统一走 getHTML。
- *来源：admin-workspace | 2026-08-12.md*

### 2. 克隆文件前检查目标结构
- **技能点**：掌握跨版本/跨目录复制文件前，必须先用 git show 查看目标文件已有同名导出与结构，再做增量合并而非整文件覆盖。
- **坑点**：直接用旧版 type.ts 覆盖新版完整类型声明文件，导致所有 QuillEditorProps 引用类型缺失，编译报 Unresolvable type reference。
- **解决方案**：保留新版原始类型声明，仅追加旧版基础工具（ToolBtn/btnStyle/processEscapeChars）；用 git show HEAD:path 提取原始内容做对比。
```text
git show HEAD:src/component/tool/quillEditorNew/type.ts > /tmp/original.ts
git show HEAD:src/component/tool/quillEditor/type.ts > /tmp/append.ts
# 保留 original 的类型声明，把 append 中的 ToolBtn/btnStyle/processEscapeChars 追加到末尾
```
- **拓展**：适用于任何"克隆/合并文件"操作，先 diff 再 merge，避免破坏目标语义。
- *来源：admin-workspace | 2026-08-12.md*

### 3. HTML 内嵌 latex 尖括号转义
- **技能点**：掌握将含裸 < > 的 LaTeX 字符串写入 innerHTML 前必须转义，避免浏览器 HTML5 解析器误判为标签。
- **坑点**：v-katex 渲染 `<question-latex>0<b<\frac{1}{2}<a<1</question-latex>` 时，裸 < 被当作标签开始，导致公式节点合并、后续内容被吞进同一公式。
- **解决方案**：正则匹配 question-latex 内容，把 < > 替换为 &lt; &gt;；取 latex 时用 textContent 自动解码回原文。
```text
html = html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi,
  (m, open, content, close) => open + content.replace(/</g, '&lt;').replace(/>/g, '&gt;') + close)
```
- **拓展**：任何"HTML 字符串中嵌入模板/公式"的场景（如富文本、Markdown 渲染）都应先转义再注入。
- *来源：admin-workspace | MEMORY.md*

### 4. Vue watch 双向同步守卫
- **技能点**：掌握 Vue 中两个 watch 互相写值导致的递归更新问题，以及用值比较守卫打破循环的方法。
- **坑点**：状态 A → 数组 → 状态 B → 状态 A 的同步链，数组引用每次变化 + deep watch 必触发 "Maximum recursive updates exceeded"。
- **解决方案**：每个 watch 内先比较推导值与当前值，相等则直接 return，不写回新引用。
```text
const next = [leftWidth, "auto", rightWidth];
if (next.every((v, i) => v === localPanelSizes.value[i])) return;
localPanelSizes.value = next;
```
- **拓展**：可沉淀为 Vue 状态同步通用模式，凡 watch 写回另一状态时都先做值比较。
- *来源：admin-workspace | 2026-08-11.md*

### 5. 按需注册组件需显式引 CSS
- **技能点**：掌握 Vue 按需注册组件（如 unplugin-vue-components + ElementPlusResolver）不等于加载组件 CSS，较新组件必须显式 import 对应 CSS。
- **坑点**：el-splitter/panel 的 CSS 未被 resolver 自动补齐，组件退化为普通 block 容器导致布局堆叠、拖拽与折叠图标全部失效。
- **解决方案**：在组件文件中显式 import 'element-plus/theme-chalk/el-splitter.css' 与 el-splitter-panel.css，逐个组件按需 import。
```text
import "element-plus/theme-chalk/el-splitter.css";
import "element-plus/theme-chalk/el-splitter-panel.css";
// 不要依赖 import 'element-plus/dist/index.css'（项目未引入）
```
- **拓展**：可推广到整个 Vue 生态：遇到组件样式异常先确认 CSS 是否真的被加载，而不是反复加 flex 掩盖。
- *来源：admin-workspace | MEMORY.md*

### 6. flex 动态高度元素对齐
- **技能点**：掌握 flex 容器中若子项包含 katex display 公式等高度不固定块级元素，必须显式设置 align-items: center 才能让兄弟行内元素垂直居中。
- **坑点**：选项 A/B/C/D 字母与 katex 公式 flex 布局时，字母默认 stretch 失败跑到公式底部，且未加 align-items 导致视觉错位。
- **解决方案**：父容器加 align-items: center；编号元素加 flex-shrink: 0 防止被挤压，必要时去掉 min-width 避免水平挤压缩放。
```text
.single-item { display: flex; align-items: center; }
.serial-number { margin-right: 4px; flex-shrink: 0; }
```
- **拓展**：可推广到任何"行内文字 + 动态高度块级内容"的 flex 布局（如头像+文本、图标+公式）。
- *来源：admin-workspace | 2026-08-12.md*

