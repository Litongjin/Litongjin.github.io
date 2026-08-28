---
title: "工作台日报 · 2026-08-29"
date: 2026-08-29 06:55:34
categories: [工作日记]
tags: ["日报", "前端", "开源", "网络安全", "AI"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-08-29

## 🔥 行业热点

- [Luanti removed from Google Play due to baseless AI copyright notice](https://blog.luanti.org/2026/08/27/luanti-dmca-tracer-ai/) — *Hacker News*
  - 📌 **内容**：开源游戏引擎 Luanti 被 Google Play 以下架通知移除，开发者认为该版权声明由 AI 生成且缺乏依据，引发对自动化版权执法风险的讨论。
  - 💡 **学习**：了解 AI 版权声明对开源项目分发的影响，掌握应对错误下架的申诉流程。
  - 🧭 **拓展**：可研究 Google Play 政策及 DMCA 反通知机制。
- [I used AWS cognito for a startup. I wouldn't do it again](https://joshkaramuth.com/blog/aws-cognito-authentication-startup-nightmare/) — *Hacker News*
  - 📌 **内容**：开发者复盘在初创项目中使用 AWS Cognito 的痛点，指出其在复杂认证场景下的局限性。
  - 💡 **学习**：评估托管认证服务与自建方案（如 Ory、Keycloak）的成本和灵活性。
  - 🧭 **拓展**：可对比 Cognito 与 Auth0、Firebase Auth 的实际体验。
- [Stopping the smart TV from being used against you](https://www.s-config.com/stopping-a-smart-tv-from-being-used-against-you/) — *Hacker News*
  - 📌 **内容**：文章介绍如何阻断智能电视的跟踪与远程控制风险，保护家庭网络安全。
  - 💡 **学习**：学习通过路由器隔离、禁用操作系统功能等手段对 IoT 设备做安全加固。
  - 🧭 **拓展**：可将类似策略应用到其他智能家居设备。
- [Autonomous Mathematical Discovery in an Open-World Multi-Agent Environment](https://arxiv.org/abs/2608.23691) — *Hacker News*
  - 📌 **内容**：研究提出在开放世界多智能体环境中实现数学自主发现，探索 AI 从数据中自行生成新定理。
  - 💡 **学习**：理解多智能体协作与自主探索在科研中的应用范式。
  - 🧭 **拓展**：可参考类似思想设计 Agent 工作流进行数据分析。
- [“It works better in the app”](https://shkspr.mobi/blog/2026/08/it-works-better-in-the-app/) — *Hacker News*
  - 📌 **内容**：文章批评了很多公司用“在 App 里体验更好”来诱导下载，忽视移动网页用户的实际需求。
  - 💡 **学习**：思考 Web 与原生 App 的体验差异，以及如何避免产品反模式。
  - 🧭 **拓展**：审查自家产品的 Web/A-B 策略，尝试用 Web 技术弥合差距。
- [GLM-5.3 is now open-weight](https://huggingface.co/zai-org/GLM-5.3) — *Hacker News*
  - 📌 **内容**：GLM-5.3 模型宣布开放权重，开发者可以下载并在私有环境中部署。
  - 💡 **学习**：利用开放权重模型进行微调、量化和本地推理，降低对闭源 API 的依赖。
  - 🧭 **拓展**：可对比该模型与其他开源模型的性能与合规性。
- [GUIs should be fully keyboard-driven](https://ckardaris.com/blog/2026/08/28/keyboard-driven-guis.html) — *Hacker News*
  - 📌 **内容**：作者主张所有 GUI 都应支持完整的键盘操作，以提升效率和无障碍性。
  - 💡 **学习**：学习键盘导航、快捷键与焦点管理的最佳实践，应用到前端开发。
  - 🧭 **拓展**：可基于 WAI-ARIA 标准评估现有应用的键盘可达性。
- [Htmx 4.0](https://four.htmx.org/announcements/2026-08-28-htmx-4.0.0-is-released) — *Hacker News*
  - 📌 **内容**：Htmx 4.0 发布，为超媒体驱动的 Web 交互带来新功能与改进。
  - 💡 **学习**：了解 Htmx 的 API 变化与新的交互能力，可用于简化前端状态管理。
  - 🧭 **拓展**：尝试用 Htmx 构建一个不依赖复杂 JS 框架的页面。
- [Inception-style curved map for turn-by-turn directions](https://www.orbify.eu/demo/) — *Hacker News*
  - 📌 **内容**：一种类似“盗梦空间”的弯曲地图 UI 设计，用于更直观地展示逐向导航。
  - 💡 **学习**：学习三维/曲线地图渲染技术，以及如何通过变形投影提高导航信息可读性。
  - 🧭 **拓展**：可以在前端可视化和地图类产品中尝试类似交互。
- [Just the rumour of a bug is enough to find an exploit these days](https://anil.recoil.org/notes/rumour-is-the-exploit) — *Hacker News*
  - 📌 **内容**：文章指出安全研究人员现在仅凭漏洞的模糊信息就能快速构造出可利用攻击。
  - 💡 **学习**：重视漏洞披露流程，理解“暗示即漏洞”对防御方的挑战。
  - 🧭 **拓展**：关注威胁情报源，及时修补同类问题。

## 🚀 技能提升点（工作总结汇总）

### 1. 克隆文件前先检查目标已有结构
- **技能点**：跨版本/跨目录合并文件时，先审计目标文件是否承载独立语义，避免盲目覆盖造成不可预期破坏。
- **坑点**：直接用旧版内容覆盖新版同名文件，导致新版完整类型声明（如 QuillEditorProps）丢失，编译报 Unresolvable type。
- **解决方案**：用 git show 查看目标文件原内容，区分“保留原结构”和“追加旧工具段”，只追加缺失导出、不替换已有结构。
```text
git show HEAD:src/component/tool/quillEditorNew/type.ts > original.ts
git show HEAD:src/component/tool/quillEditor/type.ts > extra.ts
# 保留 original.ts 完整定义，仅将 extra.ts 中的 ToolBtn/btnStyle 等追加到末尾
```
- **拓展**：可推广到任何跨版本代码合并、迁移、模块替换前的目标文件审计。
- *来源：admin-workspace 2026-08-12*

### 2. Quill 同步 v-model 用 getHTML 而非 getContents
- **技能点**：明确 vue-quill/Quill 中 getContents() 返回 Delta、getHTML() 返回 HTML，避免把 Delta JSON 字符串误当 HTML 处理。
- **坑点**：watch Quill 内容时用 getContents().trim()，得到的 Delta 对象被序列化成 JSON 后污染 v-model，DOMParser 解析后 < 被转义、data-typo-id 丢失，替换逻辑全部失配。
- **解决方案**：改为 quillRef.value?.getHTML?.() ?? ''，取真实 innerHTML；或在 watch 中用 quill.root.innerHTML。
```text
// 错误
quillRef.value?.getContents().trim()
// 正确
quillRef.value?.getHTML?.() ?? ''
```
- **拓展**：可沉淀为 Quill 编辑器封装组件的通用同步规范，防止任何 Delta/HTML 混用。
- *来源：admin-workspace 2026-08-12*

### 3. 容空白匹配解决 AI 文本与正文空格差异
- **技能点**：设计“允许任意空白”的匹配策略，在处理 AI 生成内容与已有文本比对时保持鲁棒。
- **坑点**：AI 生成的 original 与正文 latex 空格不一致（如无空格 vs \frac {\pi}{2}），严格字面匹配永远失败，导致高亮/替换缺失。
- **解决方案**：构建 whitespace-tolerant 正则（字符间 \s*），先严格匹配、未命中再容空白重试；剔除首尾空白避免吞掉公式边界。
```text
function buildWhitespaceTolerantRegex(str) {
  if (/^\s*$/.test(str)) return null;
  return new RegExp(str.split('').map(c => /\s/.test(c) ? '\\s*' : escapeRegExp(c)).join('\\s*'));
}
```
- **拓展**：可扩展为通用 fuzzy-match 工具，用于错别字、OCR 文本、AI 纠错等场景。
- *来源：admin-workspace 2026-08-12*

### 4. HTML 标签解析需引号感知与实体整体解码
- **技能点**：编写 HTML 文本提取器时，考虑属性值内裸 < > 和实体跨字符，避免标签被拦腰截断或实体被拆散。
- **坑点**：用 indexOf('>') 定位标签结束，遇到 data-formula="...\omega>0..." 的裸 > 把标签截断；逐字符解码 HTML 实体在跨字符时失败。
- **解决方案**：标签结束改为引号感知扫描（引号内跳过 >）；文本节点累积后整体 decodeHtmlEntities，并返回实体区间供偏移映射。
```text
let i = html.indexOf('<');
while (i < html.length) {
  if (html[i] === '"') { i = html.indexOf('"', i + 1); }
  else if (html[i] === '>') break;
  i++;
}
```
- **拓展**：可沉淀为通用 HTML 拍平/偏移映射工具，服务于富文本高亮、OCR 定位、diff 等。
- *来源：admin-workspace 2026-08-12*

### 5. Vue watch 双向同步必须加值比较守卫
- **技能点**：设计状态的 watched 双向同步时，先比较推导值与当前值，无变化则不再写回，防止互相触发循环。
- **坑点**：A watch 写新数组引用 → 深度 watch 触发 → 写回标志 A → 即便值未变也因引用变化再次触发，导致 Maximum recursive updates。
- **解决方案**：在每一步写入前判断当前值 !== 推导值才赋值；数组同步时先整体比较三项是否相等，相等直接 return。
```text
watch(localPanelSizes, () => {
  const next = deriveFlag();
  if (flag.value !== next) flag.value = next; // 防止无变化回写
});
```
- **拓展**：可总结为 Vue 状态同步通用规范，或封装成带 guard 的 sync helper。
- *来源：admin-workspace 2026-08-11*

### 6. Element Plus 按需注册需显式导入组件 CSS
- **技能点**：在 unplugin-vue-components + ElementPlusResolver 场景下，意识到按需注册组件不等于 CSS 已加载，新组件样式需手动补全。
- **坑点**：el-splitter 等较新组件未显式引入 CSS 时退化为普通 block，三栏布局堆叠成竖向，且拖拽/折叠图标全部失效。
- **解决方案**：在组件文件中显式 import "element-plus/theme-chalk/el-splitter.css" 和 el-splitter-panel.css，不依赖全量 index.css。
```text
import "element-plus/theme-chalk/el-splitter.css";
import "element-plus/theme-chalk/el-splitter-panel.css";
```
- **拓展**：可扩展为维护“按需导入组件所需的 CSS 清单”，所有项目通用。
- *来源：admin-workspace MEMORY.md*

