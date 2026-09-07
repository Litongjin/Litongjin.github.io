---
title: "工作台日报 · 2026-09-08"
date: 2026-09-08 07:13:39
categories: [工作日记]
tags: ["日报", "AI", "Linux", "Python", "供应链安全"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-08

## 🔥 行业热点

- [Trusting-Trust Attack against an Entire Linux Distribution](https://arxiv.org/abs/2607.24888) — *Hacker News*
  - 📌 **内容**：讨论针对整个 Linux 发行版的“信任之信任”攻击，展示构建链后门如何不留痕迹地扩散恶意代码。
  - 💡 **学习**：理解供应链攻击原理以及可信构建、重复构建与审计等防御手段。
  - 🧭 **拓展**：可尝试复现最小化信任链攻击实验，研究发行版构建的可重复性。
- [Initial effects of AI technology on employment look positive](https://www.economist.com/finance-and-economics/2026/09/04/the-jobs-apocalypse-is-postponed-an-ai-jobs-boom-is-here) — *Hacker News*
  - 📌 **内容**：报道 AI 技术对就业的初期影响，认为更多是改善而非替代。
  - 💡 **学习**：关注 AI 对开发者工作方式的改变，探索利用 AI 提升效率的新技能。
  - 🧭 **拓展**：结合自身岗位观察 AI 工具带来的效率变化，进行数据化复盘。
- [The smallest edge AI device for local LLMs](https://tiiny.ai/) — *Hacker News*
  - 📌 **内容**：介绍体积最小的边缘 AI 设备，可实现本地运行大语言模型，强调端侧推理能力。
  - 💡 **学习**：了解边缘部署 LLM 的算力约束、量化方法和模型压缩技术。
  - 🧭 **拓展**：尝试在树莓派等低功耗设备上部署小模型来体验。
- [216M Spy TVs – The LG Smart TV Problem [video]](https://www.youtube.com/watch?v=6IFVTcM28KA) — *Hacker News*
  - 📌 **内容**：揭示 LG 智能电视存在隐私风险，可能被远程监控或数据收集，涉及大量设备。
  - 💡 **学习**：认识智能设备安全风险，学习如何评估固件与通信协议中的数据泄露。
  - 🧭 **拓展**：对智能设备进行网络流量抓包分析，验证隐私行为。
- [Making a Python interpreter in 1024 bytes](https://austinhenley.com/blog/python1024.html) — *Hacker News*
  - 📌 **内容**：展示用极简代码实现 Python 解释器的技巧，挑战极小体积下的核心功能。
  - 💡 **学习**：通过极小实现理解解释器结构、词法分析和执行流程。
  - 🧭 **拓展**：阅读源码并尝试扩展支持更多语法，检验对解释器的理解。
- [Ask HN: How do you manage skills files?](https://news.ycombinator.com/item?id=49589914) — *Hacker News*
  - 📌 **内容**：HN 上关于开发者如何管理技能/知识文件的讨论，涉及个人知识管理工具与流程。
  - 💡 **学习**：学习用笔记、Git 仓库或数据库组织技能文档的方法，建立自己的知识系统。
  - 🧭 **拓展**：尝试用 Markdown + Git 搭建可版本化的技能清单。
- [Caltech Mathathon – first hackathon ever devoted to research level mathematics](https://mathathonchallenge.com/index.html) — *Hacker News*
  - 📌 **内容**：介绍 Caltech 举办的首个面向研究级数学的黑客马拉松，融合数学与编程。
  - 💡 **学习**：了解如何用计算机辅助数学研究，包括定理证明和数值实验。
  - 🧭 **拓展**：参与类似活动，或用 Python/Sage 进行数学探索。
- [WeatherNext 3](https://deepmind.google/science/weathernext/) — *Hacker News*
  - 📌 **内容**：或为新一代 AI 天气预报模型相关消息，利用深度学习提升预报精度。
  - 💡 **学习**：学习 AI 在科学计算中的应用，如图神经网络和扩散模型在时序预测中的使用。
  - 🧭 **拓展**：查阅相关论文，尝试用公开气象数据训练小型模型。
- [Simple Is Not Small](https://jyn.dev/simple-is-not-the-same-as-small/) — *Hacker News*
  - 📌 **内容**：探讨“简单”不等于代码量少，强调清晰设计比追求体积更重要的编程哲学。
  - 💡 **学习**：理解软件设计中抽象、模块化和可读性之间的平衡。
  - 🧭 **拓展**：重构一个小项目，对比简化逻辑与压缩代码的效果。
- [This Month in Ladybird – August 2026](https://ladybird.org/newsletter/2026-08-31/) — *Hacker News*
  - 📌 **内容**：Ladybird 开源浏览器项目的月度进展报告，展示浏览器引擎开发动态。
  - 💡 **学习**：关注浏览器内核的渲染、网络、JS 实现细节，了解现代浏览器架构。
  - 🧭 **拓展**：编译 Ladybird 并参与 issue 讨论，实践开源协作。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill Delta 与 HTML 同步
- **技能点**：掌握 vue-quill 的 getContents()/getHTML() API 差异：同步 v-model 或做文本处理前必须取得真实 HTML，而不是把 Delta 文档模型当 HTML 消费。
- **坑点**：误用 getContents().trim() 后用 DOMParser 解析，modelValue 被 Delta JSON 文本（含转义的 <）污染，导致后续替换/校对路径全部失配。
- **解决方案**：改为 quillRef.value?.getHTML?.() ?? ''，从 Quill 真实 innerHTML 取内容，并用 jsdom + 自定义 blot 端到端验证替换链路。
```text
// 错：getContents() 返回 Delta
const html = quillRef.value?.getContents().trim()
// 对：取真实 innerHTML
const html = quillRef.value?.getHTML?.() ?? ''
```
- **拓展**：封装任何富文本编辑器都要先厘清文档模型与渲染 HTML 两层，内容同步、脏检查、导出应统一走 HTML 通道。

### 2. HTML 属性内 LaTeX 转义与解析
- **技能点**：把 LaTeX/代码嵌进 HTML 属性前必须实体化 < >；解析 HTML 时用引号感知扫描定位标签结束，避免属性值内的原始字符截断标签。
- **坑点**：raw latex 直接塞进 data-formula 属性，flattenToPlain 用 indexOf('>') 找标签尾被属性里的 > 拦腰截断，公式内容丢失导致高亮失配；DOMParser 也会把裸 < 当标签开始。
- **解决方案**：存储侧一律转义为 &lt;/&gt;；解析侧用 findTagEnd 跳过引号内的 >，并用 [^"']* 提取属性值；替换回写时统一重编码实体。
```text
function findTagEnd(html, start) {
  let inQuote = false
  for (let i = start + 1; i < html.length; i++) {
    const code = html.charCodeAt(i)
    if (code === 0x22 || code === 0x27) inQuote = !inQuote
    if (html[i] === '>' && !inQuote) return i
  }
  return -1
}
```
- **拓展**：适用于所有将代码/公式/模板串嵌入 HTML 属性的场景；更优做法是在公共序列化层统一转义，避免各业务解析侧各自兜底。

### 3. 容空白匹配 AI 生成文本
- **技能点**：建立'严格替换优先、容空白正则兜底'的匹配策略，解决原始文本与目标正文仅空白不一致导致的失配。
- **坑点**：AI 生成的 original 与正文 latex 只差空格（一个无空格、一个有空格），escapeRegExp 严格字面匹配永远失败，表现为同批 typo 有的命中有的不命中。
- **解决方案**：用 buildWhitespaceTolerantRegex 把每字符间允许 \s*，纯空白返回 null；mark/replace 两层均先严格后宽容，collectRanges 剔除首尾空白防吞边界。
```text
function buildWhitespaceTolerantRegex(str) {
  if (!str.trim()) return null
  const parts = [...str].map(escapeRegExp)
  return new RegExp(parts.join('\\s*'))
}
// 用法：先严格，失败后宽容
const result = strictReplace(source, original, corrected) ?? tolerantReplace(source, tolerant, corrected)
```
- **拓展**：适用于错别字校对、关键词高亮、OCR 对齐、用户输入匹配等场景；正则只吞空白不吞非空白，避免跨标签误匹配。

### 4. Quill 工具栏配置 flatMap 展平
- **技能点**：理解 Quill addControls 要求单个 control 是独立对象；在可编程配置组装层对所有可能返回数组的项做扁平化，避免数组被错误解析。
- **坑点**：list/indent 的默认配置返回数组，order.map 后数组项被 Quill 当作对象以 '0' 为 format 名，生成 value=[object Object] 的 ql-0 空按钮。
- **解决方案**：将 order.map 改为 order.flatMap，数组项展开为独立 control；'formula/image' 等 handler 无条件注册，保证业务只传按钮名即可用。
```text
const controls = order.flatMap((name) => {
  const cfg = getDefaultButtonConfig(name)
  return Array.isArray(cfg) ? cfg : [cfg]
})
```
- **拓展**：动态表单、路由、菜单等配置同样存在'一项可对应多实例'的边界，统一在入口 flatten 是消除下游歧义的最稳做法。
- *来源：admin-workspace-new 2026-08-12*

### 5. Vue watch 双向同步防循环
- **技能点**：解决 Vue 中跨状态双向同步的无限递归：每次写回前比较值是否真正变化，数组引用变化不等于值变化。
- **坑点**：折叠标志 → localPanelSizes（新数组引用）→ 深 watch 回写标志，即使最终值不变也因新引用持续互相触发，报 Maximum recursive updates exceeded。
- **解决方案**：计算 next 后与当前面板尺寸逐项比较，相同则直接 return；deep watch 回写标志前也先判断 当前值 !== 推导值 再赋值，只做必要写回。
```text
const syncSizesFromFlags = () => {
  const next = [leftWidth, 'auto', rightWidth]
  if (next.every((v, i) => v === localPanelSizes.value[i])) return
  localPanelSizes.value = next
}
// deep watch 回写标志前同样先比较
if (isFilePreviewFolded.value !== deriveFolded(localPanelSizes.value[0])) {
  isFilePreviewFolded.value = deriveFolded(localPanelSizes.value[0])
}
```
- **拓展**：受控组件、表单与 store 同步、面板状态联动等所有双向 watch 都可套用'比较后写回'守卫；如可去掉双向依赖链则更佳。
- *来源：admin-workspace 2026-08-11*

### 6. 按需引入组件需显式加载 CSS
- **技能点**：排查按需注册组件（unplugin/Resolver）导致的样式失效：注册组件不等于加载对应 CSS，复杂组件要逐一验证样式存在。
- **坑点**：element-plus splitter 按需注册但 el-splitter.css 未引入，.el-splitter 退化为普通 block，方向错乱、拖拽/折叠不可见；反复改外层 flex 均无效。
- **解决方案**：在组件文件显式 import 'element-plus/theme-chalk/el-splitter.css' 与 el-splitter-panel.css；折叠图标/伪元素需覆盖默认 display:none。
```text
import 'element-plus/theme-chalk/el-splitter.css'
import 'element-plus/theme-chalk/el-splitter-panel.css'
```
- **拓展**：更普适的检查顺序：先确认根容器方向是对的，再看组件 CSS 是否被真正加载，最后才动 flex/高度，避免用样式打补丁掩盖资源缺失。
- *来源：admin-workspace MEMORY.md*

