---
title: "工作台日报 · 2026-08-30"
date: 2026-08-30 06:57:23
categories: [工作日记]
tags: ["日报", "AI Agent", "大模型", "开源", "架构"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-08-30

## 🔥 行业热点

- [Domain-Driven Agents](https://coldtake.dev/blog/domain-driven-agents) — *Hacker News*
  - 📌 **内容**：探讨把领域驱动设计（DDD）的理念引入 AI Agent 的建模与编排，让智能体围绕业务领域边界工作。
  - 💡 **学习**：可学习如何用“通用语言、限界上下文”设计 Agent 的提示词与工具调用，减少无边界幻觉行为。
  - 🧭 **拓展**：可结合实际 DDD 项目做一个事件风暴驱动的 Agent 原型。
- [StemDeck, a free, open-source and local AI stem separator](https://github.com/stemdeckapp/stemdeck) — *Hacker News*
  - 📌 **内容**：StemDeck 是一款免费、开源、可本地运行的 AI 音源分离工具，可将音乐拆分为人声/伴奏等音轨。
  - 💡 **学习**：可了解深度学习音源分离模型的本地部署与推理方法。
  - 🧭 **拓展**：可对比 Demucs 等现有方案，评估分离质量和性能。
- [Warp builds self-improving agents on Claude](https://claude.com/blog/how-warp-builds-self-improving-agents-on-claude) — *Hacker News*
  - 📌 **内容**：Warp 基于 Claude 构建能够自我改进的智能体，让 Agent 根据执行反馈修正自身行为。
  - 💡 **学习**：可学习自我反思/自我修正循环：Agent 记录任务结果并自动调整提示词或工具策略。
  - 🧭 **拓展**：可尝试在开源 Agent 框架中实现类似的经验回放机制。
- [Htmx 4.0](https://four.htmx.org/announcements/2026-08-28-htmx-4.0.0-is-released) — *Hacker News*
  - 📌 **内容**：Htmx 发布 4.0 版本，为基于 HTML 的超媒体交互带来新特性和改进。
  - 💡 **学习**：可了解 htmx 4.0 的 API 变化、扩展机制以及如何在服务端渲染中减少 JavaScript。
  - 🧭 **拓展**：可对照旧版本迁移指南，或与 Hotwire/Unpoly 等方案对比。
- [Just the rumour of a bug is enough to find an exploit these days](https://anil.recoil.org/notes/rumour-is-the-exploit) — *Hacker News*
  - 📌 **内容**：在自动化漏洞挖掘发达的当下，仅仅一个 bug 传闻就可能被安全研究者快速转化为可利用的漏洞。
  - 💡 **学习**：安全人员需要重视漏洞信息的披露时机，并建立快速响应和补丁流程。
  - 🧭 **拓展**：可研究基于 commit 消息或 issue 的自动化 PoC 生成。
- [Boot a Virtual iPhone via Apple's Virtualization.framework](https://github.com/Lakr233/vphone-cli) — *Hacker News*
  - 📌 **内容**：展示了如何借助 Apple Virtualization.framework 启动一个虚拟 iPhone 环境。
  - 💡 **学习**：可学习虚拟化框架的 API、系统镜像加载和虚拟机配置流程。
  - 🧭 **拓展**：可探索 iOS 模拟器与虚拟化的差异，或用于自动化测试。
- [The Twelve-Factor App (2025)](https://12factor.net/) — *Hacker News*
  - 📌 **内容**：十二要素应用宣言迎来 2025 年版本，继续总结云原生应用的最佳实践。
  - 💡 **学习**：可重温应用配置、进程、可移植性等十二条设计原则。
  - 🧭 **拓展**：可对照自己负责的项目逐条检查差距。
- [Tether: iMessage, SMS, etc. on Linux](https://zackbartel.com/blog/2026/08/tether/) — *Hacker News*
  - 📌 **内容**：Tether 项目试图在 Linux 上打通 iMessage、SMS 等消息服务。
  - 💡 **学习**：可学习跨平台消息桥接、协议适配和系统服务集成。
  - 🧭 **拓展**：可研究相关消息协议或 Matrix 等开放生态。
- [EVE Online moves to Python 3](https://www.eveonline.com/news/view/the-move-to-python-3-begins) — *Hacker News*
  - 📌 **内容**：大型游戏 EVE Online 将其技术栈迁移到 Python 3，是大型遗留 Python 2 代码迁移的标志性案例。
  - 💡 **学习**：可学习大型项目跨版本迁移策略、兼容层使用和逐步迁移流程。
  - 🧭 **拓展**：可参考官方迁移工具的实践经验，思考动态语言的升级治理。
- [I accidentally turned LLM memory into program analysis](https://pwning.systems/posts/llm-memory-program-analysis/) — *Hacker News*
  - 📌 **内容**：作者意外发现利用 LLM 的记忆/上下文机制可以完成程序分析任务。
  - 💡 **学习**：可探索将 LLM 的长期记忆和工具调用结合，用于代码理解、漏洞挖掘或依赖分析。
  - 🧭 **拓展**：可尝试复现并对比传统静态分析工具的结果。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill 内容同步：取 HTML 而非 Delta
- **技能点**：掌握 vue-quill/Quill 的 API 语义差异：getContents() 返回 Delta 结构化对象，getHTML()/root.innerHTML 才是可持久化的 HTML 字符串；同步 v-model 时必须用后者。
- **坑点**：误用 getContents() 返回的 Delta，再 trim 后交给 DOMParser，导致 modelValue 被 Delta JSON 字符串污染，后续公式匹配全部失败。
- **解决方案**：改用 getHTML() ?? ''（或 quill.root.innerHTML）取真实 Quill innerHTML 后同步 v-model；Delta 仅用于结构化读写。
```text
// 错误：getContents() 返回 Delta
quillRef.value?.getContents().trim();
// 正确：取真实 HTML
const html = quillRef.value?.getHTML?.() ?? quillRef.value?.root.innerHTML ?? '';
```
- **拓展**：可封装统一的 Quill 内容序列化函数（Delta→HTML），在编辑器组件中集中处理 v-model 同步。
- *来源：admin-workspace | 2026-08-12.md*

### 2. 跨版本克隆文件先查目标结构
- **技能点**：跨版本/跨目录复制文件前，先审查目标文件已有的导出与语义，采用保留原有内容 + 增量追加的合并策略，避免破坏既有类型。
- **坑点**：直接用旧版文件整体覆盖新版 type.ts，导致新版已有完整类型声明（如 QuillEditorProps）全部丢失，编译报 Unresolvable type reference。
- **解决方案**：用 git show 取目标文件原版内容作对照，保留原版完整定义，再追加旧版基础工具（ToolBtn/btnStyle 等）；不要整体覆盖。
```text
// 错误：整体覆盖丢类型
// 正确：保留原文件导出，再追加旧版片段
import type Quill from 'quill-2';
import type { QuillOptions, Delta, Range } from 'quill-2';
export interface QuillEditorProps { /* 原内容保留 */ }
export interface ToolBtn { icon: string; btnStyle?: string }
```
- **拓展**：可沉淀为迁移/重构前检查清单：先 diff 目标文件，区分底层结构与业务附加，仅做增量变更。
- *来源：admin-workspace | 2026-08-12.md*

### 3. HTML 内嵌公式的尖括号需转义
- **技能点**：掌握 HTML 内容安全：把含原始 < > 的 LaTeX 存入属性或 innerHTML 前必须先转义；解析 HTML 时不能用简单 indexOf('>') 定位标签结束。
- **坑点**：把 raw latex（如 omega>0）直接塞进 data-formula 属性，或直接 innerHTML 注入，浏览器会把内部 >/< 当成标签边界，导致内容截断或整段合并。
- **解决方案**：存储前将 < > 转义为 &lt;/&gt;，渲染用 textContent 自动解码；解析标签时用引号感知扫描跳过属性值内部字符，只在引号外找真正的 >。
```text
// 危险：属性值含 < > 破坏标签解析
el.innerHTML = `<formula data-formula="${rawLatex}">`;
// 安全：先转义
const safe = rawLatex.replace(/</g,'&lt;').replace(/>/g,'&gt;');
el.innerHTML = `<formula data-formula="${safe}">`;
```
- **拓展**：可推广到所有模板拼接 HTML 场景，统一使用实体化工具或 DOMParser，避免手写脆弱解析。
- *来源：admin-workspace | 2026-08-12.md*

### 4. Vue 双向 watch 同步加值守卫
- **技能点**：在 Vue 中做状态 ↔ 数组 ↔ 状态的双向 watch 同步时，每一步都先比较新旧值，无变化则不写，避免递归触发。
- **坑点**：watch 互相写回：标志→数组→标志，即使最终值未变，新数组引用和 deep watch 也会互相唤醒，报 Maximum recursive updates exceeded。
- **解决方案**：计算 next 后逐项比较，相同直接 return；watch 回调里也只在当前值与推导值不同时才回写。
```text
function syncSizesFromFlags() {
  const next = [leftWidth, 'auto', rightWidth];
  if (next.every((v, i) => v === localPanelSizes.value[i])) return;
  localPanelSizes.value = next;
}
watch(localPanelSizes, (val) => {
  if (isFilePreviewFolded.value !== deriveFolded(val[0])) {
    isFilePreviewFolded.value = deriveFolded(val[0]);
  }
}, { deep: true });
```
- **拓展**：可优先用 computed/单向数据流替代双向 watch；若必须同步，可封装统一的 sync 工具集中做相等判断。
- *来源：admin-workspace | 2026-08-11.md*

### 5. Quill 自定义模块需补齐默认行为
- **技能点**：掌握 Quill 模块注册机制：自定义模块接管默认能力时，必须显式补齐原默认行为，不能假定会继承。
- **坑点**：Quill.register('modules/clipboard', TableClipboard, true) 后，TableClipboard 只处理表格节点，丢失默认的 image/divider matcher，富文本图片和分割线在重新编辑时消失。
- **解决方案**：在 registerClipboardMatchers 中显式为 img[data-type="ql-image"]、divider.ql-divider 等添加 matcher，返回与对应 Blot.value 结构一致的 Delta。
```text
clipboard.addMatcher('img[data-type="ql-image"]', node => {
  const { url, alt, title, width, height, style } = node.dataset;
  return new Delta().insert({ image: { url, alt, title, width, height, style } });
});
clipboard.addMatcher('divider.ql-divider, p div hr.ql-divider, p div div.ql-divider', node =>
  new Delta().insert({ divider: { dataType: node.dataset.dataType, style: node.dataset.style } })
);
```
- **拓展**：为新增自定义 embed blot 接入剪贴板时沿用同一模式，并做 round-trip 属性对齐测试，避免恢复时丢字段。
- *来源：admin-workspace-new | MEMORY.md*

### 6. 富文本拍平与偏移映射回写
- **技能点**：在 HTML 上做关键词高亮/替换时，先把 HTML 拍平成纯文本并建立字符级偏移映射（mapIndex），在纯文本上匹配后再回写，避免破坏 HTML 结构。
- **坑点**：旧实现把标签替换成空格导致标签边界空格数不一致，跨标签 original 字面匹配失败；HTML 实体跨多字符也易被截断。
- **解决方案**：flattenToPlain 将标签映射为空串并记录映射，公式节点作为原子单元整体包裹，实体整体映射到起始偏移；匹配后按偏移回写，替换时按 typoId 合并相关片段。
```text
function flattenToPlain(html) {
  let plain = '';
  const mapIndex = [];
  // 标签→空串；文本字符逐字映射；实体整体映射到起始偏移
  return { plain, mapIndex, entityRanges };
}
// 在 plain 上得到 [start,end)，再经 mapIndex 回写到原 HTML 区间
```
- **拓展**：可沉淀为通用富文本 diff/highlight 工具，支持公式、实体、跨段混合，并补充快照回归用例。
- *来源：admin-workspace | 2026-08-12.md*

