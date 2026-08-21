---
title: "工作台日报 · 2026-08-21"
date: 2026-08-21 06:55:27
categories: [工作日记]
tags: ["日报", "AI", "AI工具", "AI编程", "Git"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-08-21

## 🔥 行业热点

- [Show HN: I trained a 125M model to autocomplete piano on-device](https://simedw.com/2026/08/20/midi-autocomplete/) — *Hacker News*
  - 📌 **内容**：作者展示了在端侧训练的1.25亿参数模型，用于钢琴音乐自动补全。
  - 💡 **学习**：可学习小模型在端侧音乐生成场景的部署与优化。
  - 🧭 **拓展**：可尝试将相同方法用于其他乐器或MIDI任务。
- [Show HN: Huzzah – a novel approach to coding with AI](https://www.danielvaughn.dev/posts/huzzah/) — *Hacker News*
  - 📌 **内容**：Huzzah 提出了一种新的 AI 辅助编程方式。
  - 💡 **学习**：关注 AI 编程工具的新交互模式与流程设计。
  - 🧭 **拓展**：可对比现有 AI 编程助手进行实践评估。
- [AliExpress runs silent WebAudio fingerprinting that breaks Bluetooth multipoint](https://blog.laserphile.com/2026/08/aliexpress-webpage-keeping-multipoint.html) — *Hacker News*
  - 📌 **内容**：报告称 AliExpress 通过 WebAudio 进行无声指纹识别，并影响蓝牙多设备连接。
  - 💡 **学习**：了解 WebAudio API 如何被用于浏览器指纹追踪及防范。
  - 🧭 **拓展**：可测试其他站点是否使用类似技术。
- [HTML Can Do That](https://chrisburnell.com/html-can-do-that/) — *Hacker News*
  - 📌 **内容**：文章介绍了一些可能被忽略的 HTML 原生能力。
  - 💡 **学习**：掌握原生 HTML 元素与属性，减少不必要的 JavaScript。
  - 🧭 **拓展**：可查阅 MDN 文档探索更多现代 HTML 特性。
- [Malicious Rust crate Arrayref runs a build-time payload](https://safedep.io/arrayref-proc-macro1-rust-build-time-malware/) — *Hacker News*
  - 📌 **内容**：Rust 库 Arrayref 被发现包含恶意构建时载荷，提醒供应链风险。
  - 💡 **学习**：构建依赖审计与供应链安全防护是重要技能。
  - 🧭 **拓展**：可使用 cargo audit 等工具检查项目依赖。
- [Mojo is now open source](https://www.modular.com/blog/mojo-open-source) — *Hacker News*
  - 📌 **内容**：编程语言 Mojo 宣布开源，开发者可获取其源码。
  - 💡 **学习**：了解 Mojo 语言的设计目标与 AI 生态集成。
  - 🧭 **拓展**：可阅读源码或尝试编译运行示例。
- [Git at any scale](https://cursor.com/blog/git-at-any-scale) — *Hacker News*
  - 📌 **内容**：讨论 Git 在不同规模代码库下的使用策略。
  - 💡 **学习**：学习大型仓库的 Git 管理技巧与性能优化。
  - 🧭 **拓展**：可尝试 monorepo 或分仓方案进行验证。
- [Stop Anthropomorphizing Intermediate Tokens as Reasoning/Thinking Traces](https://arxiv.org/abs/2504.09762) — *Hacker News*
  - 📌 **内容**：文章呼吁不要将模型中间 token 拟人化为推理过程。
  - 💡 **学习**：正确理解 LLM 内部机制，避免过度解释。
  - 🧭 **拓展**：可研究模型可解释性工具来验证中间状态。
- [Linux 7.2](https://www.igalia.com/2026/08/19/Linux-72-Released.html) — *Hacker News*
  - 📌 **内容**：Linux 内核发布 7.2 版本，带来新的特性与改进。
  - 💡 **学习**：关注内核更新中的性能与安全相关变化。
  - 🧭 **拓展**：可查看官方发布说明了解具体改动。
- [Vomit: Clean up Claude 5's token output with a separate LLM](https://github.com/zachahn/vomit) — *Hacker News*
  - 📌 **内容**：Vomit 项目利用独立 LLM 清理 Claude 5 的 token 输出。
  - 💡 **学习**：了解 LLM 输出后处理与 token 优化的思路。
  - 🧭 **拓展**：可尝试构建自定义的 LLM 输出清洗管道。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill 取内容用 getHTML 而非 getContents
- **技能点**：掌握 vue-quill/Quill 内容读取 API 差异：getContents 返回 Delta，getHTML 才返回 HTML；同步 v-model 必须取 HTML。
- **坑点**：用 getContents().trim() 把 Delta JSON 串当 HTML 写回 modelValue，导致 `<` 被 DOMParser 转义、后续匹配/替换全部失效。
- **解决方案**：统一改用 quillRef.value?.getHTML?.() ?? quillRef.value?.root?.innerHTML ?? '' 作为 Quill 内容快照。
```text
const html = quillRef.value?.getHTML?.() ?? quillRef.value?.root?.innerHTML ?? ''
```
- **拓展**：凡 Quill 内容进入 HTML 管道（v-model、表单提交、错别字匹配）都先 getHTML，并给自定义 blot 实现 getHTML 保证 round-trip。
- *来源：admin-workspace 2026-08-12*

### 2. 克隆文件前审视目标已有同名导出
- **技能点**：跨版本/跨目录合并文件时先 diff 目标文件已有导出，保留底层语义再增量追加。
- **坑点**：直接用旧版 type.ts 覆盖新版完整类型声明，导致 QuillEditorProps/Emits 等类型全部丢失、编译失败。
- **解决方案**：保留新版原始类型声明，仅追加旧版工具段（ToolBtn/btnStyle/processEscapeChars）；import type Quill 用 default 导入。
```text
import type Quill from 'quill-2';
import type { QuillOptions, Delta, Range } from 'quill-2';
```
- **拓展**：所有“克隆一个文件”的改动前先 git show 目标文件，确认无同名导出再决定覆盖或合并。
- *来源：admin-workspace 2026-08-12*

### 3. HTML 拍平匹配需维护偏移映射
- **技能点**：设计 HTML→纯文本映射（plain/mapIndex/entityRanges）以支持跨标签的定位、高亮与替换。
- **坑点**：把标签替换成空格再字面匹配，标签边界空格不一致且跨标签原文匹配失败；HTML 实体逐字符解码也拆坏实体。
- **解决方案**：标签→空串、文本节点累积后整体 decodeHtmlEntities，维护 mapIndex 与 entityRanges；匹配在纯文本上做，回写时按偏移映射回 HTML，并原子处理公式节点。
```text
function flattenToPlain(html) {
  let plain = '';
  const mapIndex = [];
  // 标签跳过，文本节点 decodeHtmlEntities 后逐字符 push mapIndex
  return { plain, mapIndex, entityRanges, formulas };
}
```
- **拓展**：该算法可沉淀为通用富文本错别字/敏感词定位与高亮工具，并支持实体、公式等原子节点。
- *来源：admin-workspace 2026-08-12*

### 4. LaTeX 空格差异需容错匹配
- **技能点**：构建 whitespace-tolerant regex：严格字面匹配优先，失败后允许字符间任意空白重试。
- **坑点**：AI 生成的 typo original 与正文 latex 空格不一致（如无空格 vs `\frac {\pi}{2}`），escapeRegExp 严格匹配永远失配。
- **解决方案**：为每个字符之间插入 `\s*` 构造容错正则，且纯空白返回 null 防死循环；匹配后剔除首尾空白，避免越界。
```text
function buildWhitespaceTolerantRegex(str) {
  const trimmed = str.trim();
  if (!trimmed) return null;
  return new RegExp([...trimmed].map(c => c === ' ' ? '\\s*' : escapeRegExp(c)).join('\\s*'));
}
```
- **拓展**：可扩展为 latex 规范化层（去空白、统一宏写法）从源头减少这类失配。
- *来源：admin-workspace 2026-08-12*

### 5. Quill 工具栏配置需 flatMap 展平
- **技能点**：理解 Quill toolbar addControls 的输入结构，确保一个按钮名可展开为多个独立 control。
- **坑点**：getDefaultButtonConfig 的 list/indent 返回数组，调用方用 map 混入数组项，Quill 把数组当对象、以索引 '0' 为 format，生成 ql-0 空按钮。
- **解决方案**：order.flatMap(name => getDefaultButtonConfig(name) ?? []) 将多按钮配置展平，每个 {list:'ordered'} 独立注册。
```text
const controls = toolbarOrder.flatMap(name => getDefaultButtonConfig(name) ?? []);
```
- **拓展**：所有“一个配置项映射多个 Quill control”的生成逻辑都统一 flatMap，并在 DOM 中检查是否出现 ql-0 异常按钮。
- *来源：admin-workspace-new 2026-08-12*

### 6. Vue watch 双向同步必须值比较守卫
- **技能点**：避免 Vue watch 之间互相触发导致递归更新：每次回写前比较值，无变化直接 return。
- **坑点**：状态A→数组新引用→状态B→状态A 的链路中，即使标志值未变，数组引用变化仍触发 deep watch，最终 'Maximum recursive updates exceeded'。
- **解决方案**：在 syncSizesFromFlags 中先比对 next 与当前数组是否相等；watch(localPanelSizes) 回写前逐个比较推导值，相等则跳过。
```text
if (next.every((v, i) => v === localPanelSizes.value[i])) return;
localPanelSizes.value = next;
```
- **拓展**：能用 computed 单向派生就不要 watch 双向同步；必须双向时先画依赖图再逐环加守卫。
- *来源：admin-workspace 2026-08-11*

