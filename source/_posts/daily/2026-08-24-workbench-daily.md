---
title: "工作台日报 · 2026-08-24"
date: 2026-08-24 06:55:59
categories: [工作日记]
tags: ["日报", "AI工具", "大模型", "AI", "AI市场"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-08-24

## 🔥 行业热点

- [I spent $266 and four AI models to own my tablet. GLM-5.3 finished it in a day](https://ericpardee.github.io/fire-hd-ownership/) — *Hacker News*
  - 📌 **内容**：作者用多个 AI 模型配合完成了对平板的完全掌控，其中 GLM-5.3 在一天内完成了关键任务。
  - 💡 **学习**：可以借鉴多模型分工与任务拆解思路，用不同 AI 处理解锁、逆向等子问题。
  - 🧭 **拓展**：可对比不同模型在真实设备破解任务上的表现，沉淀成 Agent 工作流。
- [How Complex Systems Fail (1998)](https://how.complexsystems.fail/) — *Hacker News*
  - 📌 **内容**：经典系统可靠性文章指出复杂系统的失败是常态，运维目标应是提高可恢复性而非追求绝对安全。
  - 💡 **学习**：理解复杂系统的故障模式，有助于设计更健壮的分布式系统与混沌工程实践。
  - 🧭 **拓展**：可结合故障演练和事后复盘验证文章中的模型。
- [My agent.md to improve LLM-assisted code quality](https://fabiensanglard.net/agent.md/index.html) — *Hacker News*
  - 📌 **内容**：作者通过项目级 agent.md 文件为 LLM 编程助手提供上下文与约束，从而提升 AI 辅助代码质量。
  - 💡 **学习**：可学习为 AI 编程工具编写项目说明文件，明确架构、风格与验证方式。
  - 🧭 **拓展**：可以在自己仓库中尝试维护 agent.md 并对比有/无时的代码质量。
- [Anthropic's best AI model struggles to attract users as cheaper tools thrive](https://www.ft.com/content/5ee49718-c258-4f01-aa32-7e5b76ae5245) — *Hacker News*
  - 📌 **内容**：报道 Anthropic 最强模型在用户获取上遇到挑战，而更便宜的 AI 工具正在抢占市场。
  - 💡 **学习**：关注模型能力之外的定价、速度和生态对用户选择的影响。
  - 🧭 **拓展**：可调研 API 定价与模型 benchmark 的性价比曲线。
- [AI and Infrastructure Engineering](https://omegion.dev/2026/08/ai-and-infrastructure-engineering/) — *Hacker News*
  - 📌 **内容**：探讨 AI 如何融入基础设施工程，从运维自动化到故障预测等环节。
  - 💡 **学习**：可学习用 AI 辅助监控告警、日志分析和容量规划。
  - 🧭 **拓展**：尝试在 CI/CD 或 on-call 流程中接入 LLM 助手。
- [I turned Unix talk from 1983 into the interface for my AI](https://en.andros.dev/blog/09a21bdd/i-turned-unix-talk-from-1983-into-the-interface-for-my-ai/) — *Hacker News*
  - 📌 **内容**：作者把 1983 年的 Unix talk 程序改造成与 AI 对话的终端界面，复古与新技术结合。
  - 💡 **学习**：可以学习终端 UI 和进程间通信，利用简单协议对接 AI 服务。
  - 🧭 **拓展**：参考 Unix 哲学设计轻量 AI 客户端。
- [I ran out of space extracting a RAR so I built ReclaimArc](https://github.com/harlixay7/ReclaimArc) — *Hacker News*
  - 📌 **内容**：作者因解压 RAR 时磁盘空间不足，开发了 ReclaimArc 工具来优化解压流程。
  - 💡 **学习**：可学习流式解压、磁盘空间管理及实用小工具的开发思路。
  - 🧭 **拓展**：类似思路可应用到分卷压缩、临时文件清理等场景。
- [Slovakia finds Russian backdoor in traffic speed cameras](https://risky.biz/risky-bulletin-slovakia-finds-russian-backdoor-in-traffic-speed-cameras/) — *Hacker News*
  - 📌 **内容**：斯洛伐克在交通测速摄像头中发现后门，涉及供应链与固件安全风险。
  - 💡 **学习**：提醒关注嵌入式设备固件审计与供应链安全。
  - 🧭 **拓展**：可学习固件逆向和硬件安全分析方法。
- [I gave Qwen 3.8 27B a reverse-engineering job and it finished in 30 minutes](https://www.xda-developers.com/qwen-3-8-27b-reverse-engineering-job-frontier-model/) — *Hacker News*
  - 📌 **内容**：作者用 Qwen 3.8 27B 模型做逆向工程任务，模型在 30 分钟内完成。
  - 💡 **学习**：开源大模型已能承担部分逆向工程工作，可探索其辅助二进制分析的边界。
  - 🧭 **拓展**：可以拿 CTF 样本或自有程序测试模型的逆向能力。
- [Wi-Fi 8 is the first wireless upgrade in years that isn't chasing speed](https://www.xda-developers.com/wi-fi-8-first-wireless-upgrade-years-isnt-chasing-speed-home-networks-need-it/) — *Hacker News*
  - 📌 **内容**：Wi-Fi 8 不再只追求更高速度，而是更注重稳定性、时延和能效等体验指标。
  - 💡 **学习**：了解无线协议演进方向，有助于面向新硬件的应用优化。
  - 🧭 **拓展**：可关注 Wi-Fi 8 的 MAC 层特性对物联网/实时应用的影响。

## 🚀 技能提升点（工作总结汇总）

### 1. Quill v-model 同步用 getHTML 而非 Delta
- **技能点**：掌握富文本编辑器内容与 v-model 同步时区分 Delta 与 HTML 字符串，避免错误类型污染绑定值。
- **坑点**：vue-quill 的 getContents() 返回 Delta 对象，却被当 HTML 字符串交给 DOMParser，导致 v-model 被 Delta JSON 文本覆盖，后续匹配/替换全部失效。
- **解决方案**：同步 v-model 时改用 getHTML() 或 quill.root.innerHTML 取真实 HTML；getContents() 只用于需要 Delta 的场景。
```text
// 错误：getContents() 返回 Delta
quillRef.value?.getContents().trim()
// 正确：取真实 HTML 字符串
quillRef.value?.getHTML?.() ?? ''
```
- **拓展**：所有 Quill 二次封装组件应提供 getHTML 语义的取值 API，可统一封装 serialize 函数。
- *来源：admin-workspace 2026-08-12*

### 2. HTML 纯文本提取与偏移映射
- **技能点**：掌握将 HTML 拍平为纯文本并建立原 HTML 偏移映射的算法，能处理标签、实体、原子节点等场景。
- **坑点**：标签替换为空格不折叠导致边界空格不一致；实体逐字符解码失败；data-formula 属性内含原始 < > 使标签边界扫描被截断；原始 latex 与正文空白不一致导致严格字面匹配失配。
- **解决方案**：标签映射为空串、文本节点整体解码实体、引号感知扫描标签结束、建立 mapIndex/entityRanges 偏移映射；匹配用容空白正则（字符间允许 \s*）并优先严格匹配。
```text
function findTagEnd(html, start) {
  let i = start + 1, quote = 0
  const DOUBLE = 34, SINGLE = 39
  for (; i < html.length; i++) {
    const ch = html.charCodeAt(i)
    if (quote) {
      if (ch === quote) quote = 0
    } else if (ch === DOUBLE || ch === SINGLE) {
      quote = ch
    } else if (ch === 62) { // >
      break
    }
  }
  return i
}
```
- **拓展**：可沉淀为通用 html2plain + offsetMap 工具，供高亮、错别字替换、diff、搜索等复用。
- *来源：admin-workspace 2026-08-12*

### 3. 克隆文件前检查目标已有结构
- **技能点**：建立先看目标再动手的代码合并习惯，避免用旧版内容覆盖承载语义的新版文件。
- **坑点**：直接拿旧版 type.ts 覆盖新版 type.ts，导致新版完整类型声明被抹掉，所有引用处类型解析失败。
- **解决方案**：若目标文件已存在且承载结构/类型，只追加旧版基础工具段，保留新版原始声明；用 git show 取原始内容再合并。
- **拓展**：可推广到任何跨版本/跨分支代码同步：先 diff 目标文件，识别语义差异，再决定 clone 还是 merge。
- *来源：admin-workspace 2026-08-12*

### 4. Vue watch 双向同步防循环
- **技能点**：掌握 Vue 中多状态双向同步的防递归技巧，理解引用变化与 deep watch 的组合效应。
- **坑点**：状态 A 到数组再到状态 B 再到状态 A 的同步链中，每次写新数组引用触发 deep watch，即使值未变也继续回写，导致 Maximum recursive updates。
- **解决方案**：在每一步写回前先比较当前值与推导值，相等则直接 return；避免监听高频变化的 props 引用，改在真正的事件 handler 中触发同步。
```text
watch(localPanelSizes, (v) => {
  const next = deriveFlag(v)
  if (flag.value !== next) flag.value = next
}, { deep: true })
```
- **拓展**：可封装 compare-and-set 的 ref 赋值工具，或在 watch 回调统一用 hasChanged 守卫。
- *来源：admin-workspace 2026-08-11*

### 5. 按需注册组件需显式加载 CSS
- **技能点**：理解 unplugin-vue-components 等按需注册只处理 JS 注册，不会自动补齐组件样式，养成显式 import CSS 的习惯。
- **坑点**：ElementPlusResolver 按需注册了 el-splitter，但未加载对应 CSS，组件退化为普通 block 布局，面板上下堆叠。
- **解决方案**：显式 import element-plus/theme-chalk/el-splitter.css 等；不要依赖全量 index.css（项目可能未引入）。
```text
import 'element-plus/theme-chalk/el-splitter.css'
import 'element-plus/theme-chalk/el-splitter-panel.css'
```
- **拓展**：对任何按需引入的组件库，遇到样式/布局异常先检查对应 CSS 是否被加载，再排查业务样式。
- *来源：admin-workspace MEMORY.md*

### 6. Quill 工具栏数组配置需展平
- **技能点**：掌握 Quill toolbar 配置项的数据结构要求，避免数组被当成对象控件解析。
- **坑点**：getDefaultButtonConfig 对 list/indent 返回数组，buildToolbarContainer 未展平，Quill addControls 把数组当对象，生成 format=0、value=[object Object] 的 ql-0 空按钮。
- **解决方案**：用 flatMap 将多按钮数组展平为独立 control 项，确保每个 control 都是 { format: value } 结构。
```text
const controls = toolbarOrder.flatMap(name => getDefaultButtonConfig(name) ?? [])
```
- **拓展**：可补充 Quill 工具栏配置的运行时校验，检测非对象 control 并报错。
- *来源：admin-workspace-new 2026-08-12*

### 7. 自定义 clipboard 需补齐默认 matcher
- **技能点**：掌握 Quill 模块覆写后需显式恢复默认行为，避免功能静默丢失。
- **坑点**：注册 TableClipboard 接管 clipboard 后，它没有继承默认的 image/divider matcher，convert 时图片和分割线被丢弃。
- **解决方案**：在 registerClipboardMatchers 中显式 addMatcher 处理 img[data-type=ql-image] 和 divider.ql-divider，并确保属性结构与对应 Blot.value 对齐。
```text
clipboard.addMatcher('img[data-type="ql-image"]', node => {
  const img = node as HTMLElement
  return new Delta().insert({ image: { url: img.getAttribute('src'), alt: img.getAttribute('alt') } })
})
```
- **拓展**：新增自定义 embed blot 时同样显式加 matcher，不假设自定义模块会兜底。
- *来源：admin-workspace-new MEMORY.md*

