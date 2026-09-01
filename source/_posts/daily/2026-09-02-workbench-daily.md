---
title: "工作台日报 · 2026-09-02"
date: 2026-09-02 07:01:49
categories: [工作日记]
tags: ["日报", "前端", "大模型", "开源", "AI Agent"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-02

## 🔥 行业热点

- [I trained a small transformer in 1.5hrs and it beats many LLMs](https://mvakde.github.io/blog/44-on-arc-1/) — *Hacker News*
  - 📌 **内容**：作者用很短时间训练了一个小型Transformer，并在多项任务上超过许多大模型，展示小模型高效训练的可行性。
  - 💡 **学习**：可关注小模型的数据配比、训练时长与蒸馏技术，思考如何用更低成本逼近大模型效果。
  - 🧭 **拓展**：可复现类似实验，在自己的数据集上比较小模型与API大模型的效果。
- [Keenable SELECT: an agent that searches the web in SQL](https://keenableai.github.io/select-showcase/) — *Hacker News*
  - 📌 **内容**：该工具把网络搜索抽象为SQL查询，让开发者可以用声明式语法驱动AI Agent检索信息。
  - 💡 **学习**：可学习如何将外部API/搜索引擎封装成SQL接口，降低Agent调用成本。
  - 🧭 **拓展**：可尝试在自己项目中构建类似的统一查询层。
- [Show HN: Weedout – Safari extension that hides YouTube AI-labeled videos](https://masteranza.github.io/weedout/) — *Hacker News*
  - 📌 **内容**：一个Safari扩展，用于隐藏YouTube上带有AI生成标签的视频，帮助用户过滤内容。
  - 💡 **学习**：可学习Safari扩展开发、内容脚本注入和基于标签的过滤逻辑。
  - 🧭 **拓展**：可扩展为支持其他平台或自定规则的内容过滤器。
- [Play Store blocks AuroraStore, hurting GrapheneOS users](https://gitlab.com/AuroraOSS/AuroraStore/-/work_items/1566) — *Hacker News*
  - 📌 **内容**：Google Play对AuroraStore的阻止影响了GrapheneOS用户的App获取，引发对Android生态开放性的讨论。
  - 💡 **学习**：可了解AuroraStore与Play Store的交互机制，以及无GMS设备的应用分发方案。
  - 🧭 **拓展**：可研究GrapheneOS环境下应用商店的替代实现。
- [Introducing Ad Blocker for Firefox on iOS](https://blog.mozilla.org/en/firefox/ad-blocker-on-ios/) — *Hacker News*
  - 📌 **内容**：Firefox在iOS平台上线了广告拦截功能，为Safari之外提供更多内容过滤选择。
  - 💡 **学习**：可学习iOS上基于WebKit内容拦截器的广告拦截实现方式。
  - 🧭 **拓展**：可对比Firefox与Safari扩展API的差异。
- [The ChatGPT/Codex app bundles a full copy of LibreOffice](https://simonwillison.net/2026/Sep/1/codex-libreoffice/) — *Hacker News*
  - 📌 **内容**：ChatGPT/Codex桌面应用中内置了完整LibreOffice，可能用于文档处理与导出。
  - 💡 **学习**：可关注AI应用如何集成办公套件，以及文档格式兼容的实现思路。
  - 🧭 **拓展**：可尝试用LibreOffice命令行完成文档转换任务。
- [Ambient CSS v3 – Blender meets CSS](https://ambientcss.vercel.app/) — *Hacker News*
  - 📌 **内容**：Ambient CSS v3将类似Blender的3D创作体验带入CSS，让网页能更轻松生成三维界面。
  - 💡 **学习**：可学习CSS 3D变换、Web图形渲染和前端程序化建模。
  - 🧭 **拓展**：可创建3D交互页面并对比Three.js等方案。
- [The creator of Jujutsu has joined ERSC](https://ersc.io/blog/martin-joins-ersc) — *Hacker News*
  - 📌 **内容**：版本控制工具Jujutsu的创建者加入ERSC，可能推动该工具及其理念的进一步演进。
  - 💡 **学习**：可关注Jujutsu的设计思路，以及它相对Git的工作流改进。
  - 🧭 **拓展**：可尝试用Jujutsu管理项目并对比Git体验。
- [Movie Scene Map – 13,312 films, series, games, anime and manga](https://moviescenemap.com/) — *Hacker News*
  - 📌 **内容**：该项目将13,312部影视、游戏和动漫的场景地点映射到地图，形成大规模文化地理数据。
  - 💡 **学习**：可学习如何构建大规模地图可视化、实体识别与坐标标注 pipeline。
  - 🧭 **拓展**：可基于这些数据做地理分布或类型偏好分析。
- [Show HN: Running 104GB Qwen3.8-Flash-Next on 48GB Mac with at ~12 tok/s](https://github.com/carloslfu/slotstream) — *Hacker News*
  - 📌 **内容**：作者在48GB内存的Mac上运行104GB规模的Qwen模型，并通过优化达到约12 tok/s。
  - 💡 **学习**：可学习模型量化、权重卸载和内存复用等端侧推理优化技巧。
  - 🧭 **拓展**：可复现该环境测试其他大模型的本地部署效果。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill getContents 与 getHTML 混用
- **技能点**：掌握 vue-quill 中 Delta 与 HTML 的区分，同步 v-model 时必须取真实 HTML。
- **坑点**：watch 里用 getContents() 返回 Delta，被当 HTML 字符串解析，导致 modelValue 被 Delta JSON 污染，后续匹配全部失败。
- **解决方案**：改用 quillRef.getHTML() ?? '' 获取真实 innerHTML，再走后续处理。
```text
const html = quillRef.value?.getHTML?.() ?? ''
```
- **拓展**：所有 Quill 封装组件在同步 v-model 前应建立统一 getHTML 出口，避免多处重复踩坑。
- *来源：admin-workspace-new，2026*

### 2. HTML 解析对属性中裸 < > 的容忍
- **技能点**：掌握自写 HTML 解析器时对属性值的引号感知扫描，避免被数据中的 < > 截断标签。
- **坑点**：latex 中的 < / > 未实体化直接塞进 data-formula，flattenToPlain 用 indexOf('>') 定位标签结束，公式内容被拦腰截断丢失。
- **解决方案**：标签结束改为引号感知：进入引号内的 > 跳过，直到引号外遇到真正的 > 才结束；数据侧也应将 latex 的 < > 实体化。
```text
let inQuote = false; for (; i < html.length; i++) { const ch = html[i]; if (ch === '"' && html[i-1] !== '\\') inQuote = !inQuote; if (ch === '>' && !inQuote) break; }
```
- **拓展**：任何手写 HTML 解析都要考虑属性值内含特殊字符，优先用 DOM parser 或规范转义。
- *来源：admin-workspace，2026*

### 3. 文本匹配容空白差异
- **技能点**：掌握构建容错正则匹配用户输入与正文之间的空白不一致。
- **坑点**：AI 生成 latex 与正文 latex 空格不一致，严格字面匹配永远失配，导致高亮/替换不生效。
- **解决方案**：新增 buildWhitespaceTolerantRegex 在每个字符间允许 \s*，严格匹配失败时再走容空白重试；注意纯空白返回 null 防死循环。
```text
const buildWhitespaceTolerantRegex = (s) => s.trim() ? new RegExp(s.split('').map(c => /\s/.test(c) ? '\\s*' : c).join('\\s*')) : null;
```
- **拓展**：可推广到任何由 AI 生成文本与原文做匹配的场景，同时注意正则性能与误匹配边界。
- *来源：admin-workspace，2026*

### 4. Quill 工具栏配置数组必须展平
- **技能点**：掌握 Quill toolbar 配置中数组项需 flatMap 展平，避免对象被错误解析。
- **坑点**：getDefaultButtonConfig 返回数组（list/indent 多按钮），order.map 后数组项被当作对象，Object.keys 取到 '0' 生成 ql-0 空按钮。
- **解决方案**：调用方把 map 改为 flatMap，使每个 {list:'ordered'} 独立成 control。
```text
const mapped = order.flatMap(name => getDefaultButtonConfig(name));
```
- **拓展**：在封装任何带分组配置的库时，需对 order 进行展平处理并补充单测。
- *来源：admin-workspace-new，2026-08-12*

### 5. Vue watch 双向同步死循环
- **技能点**：掌握 Vue watch 双向同步的值比较守卫，防止无限递归。
- **坑点**：折叠标志与 panel sizes 互相 watch，每次写新数组/新值都触发对端 watch，即使值未变也因引用变化继续循环，报 Maximum recursive updates。
- **解决方案**：每个 watch 回调内先比较新旧值，无变化直接 return；写状态前也判等。
```text
if (next.every((v, i) => v === localPanelSizes.value[i])) return;
```
- **拓展**：任何 A→B→A 的 watch 链都要加守卫；也可用单一数据源合并为单向推导。
- *来源：admin-workspace，2026-08-11*

### 6. flex 子项对 katex 公式垂直对齐
- **技能点**：掌握 flex 布局中不定高块级子项的对齐处理。
- **坑点**：选项编号与 katex 显示公式在同一 flex 行，缺省 align-items 导致编号贴底。
- **解决方案**：父容器设 align-items: center，编号加 flex-shrink: 0 与 margin-right。
```text
.single-item { display: flex; align-items: center; } .serial-number { flex-shrink: 0; margin-right: 4px; }
```
- **拓展**：在富文本/公式混排场景，应显式定义行内元素与块级公式的垂直对齐策略。
- *来源：admin-workspace，2026*

### 7. 关闭按需组件 CSS 缺失排查
- **技能点**：掌握 element-plus 按需注册时组件样式未自动加载的排查路径。
- **坑点**：el-splitter 等新组件 CSS 未随 resolver 自动引入，布局退化为普通块级上下堆叠。
- **解决方案**：在组件内显式 import 'element-plus/theme-chalk/el-splitter.css' 与 el-splitter-panel.css。
```text
import 'element-plus/theme-chalk/el-splitter.css';
import 'element-plus/theme-chalk/el-splitter-panel.css';
```
- **拓展**：凡是使用按需注册组件库时，遇到布局异常先检查对应 CSS 是否真的被加载。
- *来源：admin-workspace，MEMORY.md*

