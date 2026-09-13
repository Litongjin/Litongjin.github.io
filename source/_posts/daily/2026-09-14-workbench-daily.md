---
title: "工作台日报 · 2026-09-14"
date: 2026-09-14 07:02:36
categories: [工作日记]
tags: ["日报", "Rust", "大模型", "AI基础设施", "AI安全"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-09-14

## 🔥 行业热点

- [Why are AI agents lying, cheating and coordinating?](https://yoshuabengio.org/en/publication/why-are-ai-agents-lying-cheating-and-coordinating) — *Hacker News*
  - 📌 **内容**：探讨AI智能体在协作与目标追求中出现的说谎、作弊等行为及其成因。
  - 💡 **学习**：可关注多智能体系统中行为安全与评估方法的设计。
  - 🧭 **拓展**：可结合主流Agent框架模拟协作场景进行验证。
- [Nvidia is the central bank of AI](https://www.economist.com/interactive/briefing/2026/09/03/nvidia-is-the-central-bank-of-ai) — *Hacker News*
  - 📌 **内容**：将Nvidia类比为AI世界的中央银行，强调其算力供给对AI产业的决定性影响。
  - 💡 **学习**：可理解算力供需关系对模型训练成本与应用生态的影响。
  - 🧭 **拓展**：可跟踪GPU市场价格与云厂商定价变化。
- [Garry Tan wants US open-weight AI labs to 'distill' frontier models, too](https://techcrunch.com/2026/09/11/y-combinators-garry-tan-wants-u-s-open-weight-ai-labs-to-distill-frontier-models-too/) — *Hacker News*
  - 📌 **内容**：讨论美国开放权重AI实验室对更强模型进行蒸馏的呼声，涉及开源与前沿模型迭代方式。
  - 💡 **学习**：可理解蒸馏技术在开源模型追赶中的角色。
  - 🧭 **拓展**：可关注相关实验室的模型发布策略。
- [TailTalk: A modern async user space AppleTalk stack with Rust and Tokio](https://github.com/FeralFirmware/TailTalk/) — *Hacker News*
  - 📌 **内容**：介绍用Rust与Tokio实现的全新异步用户态AppleTalk协议栈。
  - 💡 **学习**：可学习Rust异步网络编程及协议栈从零实现的方法。
  - 🧭 **拓展**：可阅读其源码并与经典AppleTalk实现对比。
- [Make your first edit to OpenStreetMap](https://high5apps.github.io/josm-plugin-website-wizard/) — *Hacker News*
  - 📌 **内容**：面向新人介绍如何从零开始参与OpenStreetMap地图编辑。
  - 💡 **学习**：可学习OSM的数据模型、编辑工具与开放数据协作流程。
  - 🧭 **拓展**：可选取本地街区完成一次真实编辑。
- [Homebrew 7.0.0](https://brew.sh/2026/09/13/homebrew-7.0.0/) — *Hacker News*
  - 📌 **内容**：macOS包管理器Homebrew发布7.0.0正式版。
  - 💡 **学习**：可查看更新日志了解新的命令行行为与兼容性变化。
  - 🧭 **拓展**：可升级并验证项目依赖是否正常。
- [JetKVM Mini](https://jetkvm.com/blog/introducing-jetkvm-mini) — *Hacker News*
  - 📌 **内容**：介绍JetKVM Mini这一KVM-over-IP远程管理硬件设备。
  - 💡 **学习**：可了解远程服务器控制与带外管理方案。
  - 🧭 **拓展**：可评测其网络配置与安全性。
- [Astra and Fable still hack on simple variants of alignment evals from 2025](https://www.lesswrong.com/posts/munJKF7iWMsWJLAH2/astra-and-fable-still-hack-on-simple-variants-of-alignment) — *Hacker News*
  - 📌 **内容**：指出Astra和Fable仍在迭代2025年的简单对齐评估变体，关注AI安全评测的可持续性。
  - 💡 **学习**：可学习如何设计轻量可迭代的alignment评估任务。
  - 🧭 **拓展**：可参考其思路完善自己的模型评测集。
- [Reverse engineering my e-scooter and rewriting the firmware in Rust](https://bensimms.moe/reverse-engineering-scooter/) — *Hacker News*
  - 📌 **内容**：记录逆向电动滑板车控制器并用Rust重写固件的过程。
  - 💡 **学习**：可学习嵌入式逆向、固件分析与Rust在裸机环境的应用。
  - 🧭 **拓展**：可参考其硬件调试方法应用到其他IoT设备。
- [Data collected by cars and sold to third parties](https://www.theverge.com/column/994172/your-car-is-selling-your-data) — *Hacker News*
  - 📌 **内容**：关注汽车采集用户数据并出售给第三方的隐私风险。
  - 💡 **学习**：可了解车载数据生态、数据合规与隐私保护技术。
  - 🧭 **拓展**：可调研主流车企的数据共享政策。

## 🌟 GitHub 热门开源项目

> 📈 本周候选 84 个仓库中「Agent Skills」占 29 个，是当前最活跃赛道。

- [browser-use/browser-use（Agent 框架）](https://github.com/browser-use/browser-use) — *GitHub · Python · 周增量统计中 · 总 114.5k star*
  - 📌 **是什么**：让 LLM agent 通过 Playwright 控制浏览器完成网页操作的开源库，实现从自然语言到浏览器行为的端到端自动化。
  - 💡 **学习点**：可学习如何将大模型输出与浏览器自动化工具结合，构建能真实“使用”网页的 agent。
  - 🧭 **上手**：运行一个快速示例，观察 agent 如何将任务拆解成浏览器动作序列。
- [google-gemini/gemini-cli（MCP 工具）](https://github.com/google-gemini/gemini-cli) — *GitHub · TypeScript · 周增量统计中 · 总 107.0k star*
  - 📌 **是什么**：在终端中运行的 Gemini 驱动 AI agent，集成了 MCP 客户端与服务器能力。
  - 💡 **学习点**：适合学习如何构建 CLI 形态的 agent，并理解 MCP 协议在终端工具中的双向集成。
  - 🧭 **上手**：在终端执行一个任务并开启 MCP 配置，查看它与外部工具交互的流程。
- [nexu-io/open-design（Agent Skills）](https://github.com/nexu-io/open-design) — *GitHub · TypeScript · 周增量统计中 · 总 95.9k star*
  - 📌 **是什么**：面向 DeepSeek 等编码 agent 的设计插件，让 AI agent 直接生成原型、落地页和仪表盘等设计产物。
  - 💡 **学习点**：学习如何为编码 agent 添加设计技能，使 AI 从写代码扩展到产出视觉界面。
  - 🧭 **上手**：安装插件后用 coding agent 生成一个简单页面，观察设计到代码的转换链路。
- [addyosmani/agent-skills（Agent Skills）](https://github.com/addyosmani/agent-skills) — *GitHub · JavaScript · 周增量统计中 · 总 94.0k star*
  - 📌 **是什么**：面向 AI 编码 agent 的生产级工程技能集合，覆盖代码审查、重构等最佳实践。
  - 💡 **学习点**：了解如何把工程经验沉淀为 agent 可复用的 skill 文件，提升 AI 编码质量。
  - 🧭 **上手**：阅读其中一个 skill 文件，看它的触发条件和执行步骤是如何组织的。
- [thedotmack/claude-mem（Agent Skills）](https://github.com/thedotmack/claude-mem) — *GitHub · TypeScript · 周增量统计中 · 总 93.8k star*
  - 📌 **是什么**：为 Claude 等 agent 实现跨会话持久记忆的工具，自动压缩会话内容并在未来对话中注入相关上下文。
  - 💡 **学习点**：学习如何设计 agent 记忆层，用压缩和检索解决上下文窗口限制。
  - 🧭 **上手**：跑一个会话后查看记忆文件，理解信息抽取与注入机制。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步死循环
- **技能点**：掌握 Vue 中多个 watch 互相触发导致无限递归的排查与防御，学会用值比较守卫打破循环。
- **坑点**：状态A→数组→状态B→状态A 的同步链中，每次写新数组引用（即使值相同）触发 deep watch，又回写标志，反复唤醒 watch，报 Maximum recursive updates。
- **解决方案**：在每条同步分支写回前比较新旧值，相等直接 return 不写引用；对标志回写前也先判断 当前值 !== 推导值 再赋值。
```text
const next = [leftWidth, "auto", rightWidth];
if (next.every((v, i) => v === localPanelSizes.value[i])) return;
localPanelSizes.value = next;
if (isFilePreviewFolded.value !== derived) isFilePreviewFolded.value = derived;
```
- **拓展**：可推广到任意响应式多状态互相派生的场景，优先设计单一数据源；用 watchEffect 时也需同样的守卫。
- *来源：admin-workspace 2026-08-11*

### 2. Quill 编辑器 v-model 初始化时序
- **技能点**：理解子组件 setup 阶段 watch immediate 与实例未就绪的时序问题，学会用待写入队列缓存初始值。
- **坑点**：watch(props.modelValue, ..., { immediate: true }) 在 quillInstance 创建前立即触发，直接 return 导致弹窗首次打开时编辑内容空白。
- **解决方案**：增加 pendingModelValue 队列缓存未就绪时的值；initQuill 后的 nextTick 阶段 flushPendingModelValue()；业务侧弹窗打开 nextTick + requestAnimationFrame 调 setContent 双保险。
```text
let pendingModelValue: string | null = null;
watch(() => props.modelValue, v => {
  if (!quillInstance) { pendingModelValue = v ?? null; return; }
  if (v !== undefined) quill.setContents(...);
}, { immediate: true });

function flushPendingModelValue() {
  if (pendingModelValue != null) { quill.setContents(...); }
}
```
- **拓展**：适用于所有“实例异步创建、props 同步到达”的组件封装（富文本、地图、图表等），可统一成 useAsyncInstance 模式。
- *来源：admin-workspace-new MEMORY.md*

### 3. Quill 自定义 clipboard matcher 补齐
- **技能点**：掌握 Quill clipboard 模块在自定义注册后不会继承默认 matcher，需为自定义 embed 显式补充解析器。
- **坑点**：用 TableClipboard 接管 clipboard 后，convert 时 image/divider 等默认 matcher 丢失，富文本编辑弹窗里图片和分割线全没了（文字正常）。
- **解决方案**：在 registerClipboardMatchers 中显式 addMatcher：img[data-type="ql-image"] → Delta.insert({image})、divider.ql-divider → Delta.insert({divider})；属性结构需与对应 Blot.value 对齐。
```text
clipboard.addMatcher('img[data-type="ql-image"]', node => {
  const { url, alt, title, width, height, style } = (node as HTMLElement).dataset;
  return new Delta().insert({ image: { url, alt, title, width, height, style } });
});
clipboard.addMatcher('divider.ql-divider, p div hr.ql-divider, p div div.ql-divider', node =>
  new Delta().insert({ divider: { dataType: (node as HTMLElement).dataset.dataType, style: (node as HTMLElement).getAttribute('style') } })
);
```
- **拓展**：新增自定义 embed blot 时沿用同一模式，不假设 TableClipboard 会兜底；可沉淀为 Quill 封装组件的 matcher 清单。
- *来源：admin-workspace-new MEMORY.md*

### 4. 按需组件注册不等于加载 CSS
- **技能点**：熟悉 unplugin-vue-components + ElementPlusResolver 场景下，新组件 CSS 可能未自动引入，需要显式 import 组件 CSS。
- **坑点**：el-splitter 三栏布局退化为 block 流式堆叠，因为项目未全量引入 element-plus CSS，resolver 未补齐 splitter 相关样式。
- **解决方案**：在组件 index.vue 显式 import "element-plus/theme-chalk/el-splitter.css" 与 el-splitter-panel.css；排查布局先看根容器方向，再看组件 CSS 是否真的被加载。
```text
import "element-plus/theme-chalk/el-splitter.css";
import "element-plus/theme-chalk/el-splitter-panel.css";
```
- **拓展**：适用于 element-plus 较新组件（splitter、virtual-list 等）和其他组件库的按需引入场景；建立“用新组件先查 CSS 是否加载”的检查清单。
- *来源：admin-workspace MEMORY.md*

### 5. LaTeX 裸尖括号 HTML 转义
- **技能点**：明确 innerHTML 赋值会让浏览器 HTML5 解析器将裸 < 当标签开始，需先转义再渲染，并用 textContent 解码还原。
- **坑点**：v-katex 渲染 <question-latex>0<b<...<a<1</question-latex> 时，latex 内裸 < 与 > 破坏闭合标签，整段文字被合并进第一个公式，公式节点从 41 变 3。
- **解决方案**：用正则 /(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi 将内容中的 < > 转义为 &lt; &gt; 后再 innerHTML；getLatex 用 textContent 自动解码回原 latex。
```text
html = html.replace(/(<question-latex\b[^>]*>)([\s\S]*?)(<\/question-latex>)/gi, (_, s, body, e) => {
  return s + body.replace(/</g, '&lt;').replace(/>/g, '&gt;') + e;
});
el.innerHTML = html;
```
- **拓展**：任何渲染接口/用户输入的数学模板文本都应预转义，可封装为 sanitizeLatex 工具并配 jsdom 回归测试。
- *来源：admin-workspace 2026-08-10*

### 6. antd 受控 value 对数字 0 的 falsy 丢失
- **技能点**：识别受控组件用 `value || undefined` 会吞掉 0，学会用字符串 value 保存并 Number() 转回。
- **坑点**：SelectFilter 用 value={config.value || undefined}，筛选项 “第1轮/0号” 等 value 为 0 时被当作 falsy，选项无法回显。
- **解决方案**：涉及从 0 开始的语义（轮次、序号）时 options value 用字符串 '0'/'1'，onChange 里 Number(value) 转回数字。
```text
<Select
  value={currentValue === undefined ? undefined : String(currentValue)}
  onChange={v => onChange(Number(v))}
/>
```
- **拓展**：同类陷阱存在于 Tabs activeKey、Radio、Switch 等所有受控值；可统一约定“有 0 语义的选项 value 用字符串”。
- *来源：admin-workspace-hr-talent MEMORY.md*

### 7. antd Table sticky 与 scroll.x 的配合
- **技能点**：区分 antd sticky prop（横向 sticky-scroll）与“表头吸顶”需求，知道 sticky 必须配 scroll.x 否则表头列错位。
- **坑点**：启用 sticky 不配 scroll.x 时表头与 body 右侧列错位；sticky 还会自动启用横向 sticky-scroll 滚动条，列少时出现横滚条+右侧空白。
- **解决方案**：需要 sticky 时同时配 scroll={{ x: 'max-content' }}；仅表头吸顶+纵向滚动时不用 sticky prop，改 CSS `.xxx-table .ant-table-thead > tr > th { position: sticky; top: 0; z-index: 2 }`。
```text
.xxx-table .ant-table-thead > tr > th {
  position: sticky;
  top: 0;
  z-index: 2;
}
```
- **拓展**：scroll.x 一律 'max-content' 避免写死列宽过期；CSS sticky 方案在最近滚动祖先内生效，无副作用，可作为列表吸顶的统一模式。
- *来源：admin-workspace-hr-talent MEMORY.md*

