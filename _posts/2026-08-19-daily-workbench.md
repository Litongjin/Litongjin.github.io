---
title: "工作台日报 · 2026-08-19"
date: 2026-08-19 12:09:11 +0800
categories: [日报]
author: Litongjin
---

# 工作台日报 · 2026-08-19

> 自动生成于 2026-08-19 12:09 · 个人工作台 Agent

## 🔥 行业热点

- [Using the railway network as a flatbed scanner](https://philo.gay/linecam/) — *Hacker News*
- [AI usage patterns in software teams](https://linear.app/data) — *Hacker News*
- [The Amazon tax](https://seths.blog/2026/08/the-amazon-tax/) — *Hacker News*
- [Universal health coverage could save $1T and 114k lives a year: study](https://ysph.yale.edu/news-article/universal-health-coverage-could-save-one-trillion-dollars-and-114000-lives-every-year/) — *Hacker News*
- [Memory prices climb 500% in 12 months](https://www.tomshardware.com/pc-components/ram/memory-prices-climb-500-percent-in-12-months-up-to-10x-the-lowest-ever-tracked-prices-128gb-of-ddr5-now-usd3-399) — *Hacker News*
- [Cursor launches Origin, GitHub alternative](https://cursor.com/changelog/origin-code-hosting) — *Hacker News*
- [Beware Management Consultants](https://about.iceland.co.uk/our-story/the-dark-ages/beware-management-consultants/) — *Hacker News*
- [Fixing a bricked Framework laptop](https://quantum5.ca/2026/08/16/fixing-bricked-amd-7040-series-framework-13-laptop-with-20-tools/) — *Hacker News*
- [Being ambitious and being a dad](https://nicholascharriere.com/blog/being-ambitious-and-being-a-dad/) — *Hacker News*
- [New paper shows that 37% of workers in US saw real wages decline from 2021-2024 [pdf]](https://bfi.uchicago.edu/wp-content/uploads/2026/08/BFI_WP_2026-108-1.pdf) — *Hacker News*

## 🚀 技能提升点（工作总结汇总）

### 1. 错别字匹配：纯文本拍平匹配 + 偏移回写方案落地
- **坑点**：旧实现 `text.replace(/<[^>]*>/g, " ")` 把标签替换成空格且不折叠，导致标签边界空格数不一致（`设 </span>y` → `设 y` ≠ original `设 y`），且跨标签 original 字面匹配失败，真实题干高亮失配。
- **解决方案**：截图"红框不完整"根因在替换阶段：`replaceInFieldWithNoStyle` 第 2 步正则要求 `>${escapedOriginal}</span>`（字面完整 original），但跨段 typo 每个 span 只含 original 一部分，整段匹配永远失败 → 文字段全漏替换。 修复（`questionCheck.ts`）： - `r
- **拓展**：#0/#2 original 与正文结构/粒度不匹配：#0 的 `/\frac` 是残缺式、正文是已渲染公式；#2 original 是公式子串。需后端校准 original（指向完整公式节点或对齐完整公式）。
- *来源：工作总结*

### 2. 修复：paperAI审核——翻页后"暂无数据"+失败题不显示（已落地）
- **坑点**：现象：`auditStatus:"40"`（FAILED）的题目既不出现在题号 tag 也不显示内容。 - 根因：`processedQuestions` 第 338 行 `auditStatus !== AuditStatusEnum.FAILED` 直接被过滤掉。 - 修复（`aiCheckGroupQuestion.vue`）： - `processe
- **解决方案**：旧：`emit('refresh')` → 父 `handleAiRefresh` → `handleAiReviewer(true)` 会把整卷 allQuestionIds 重新提交审核，且有 ElMessageBox 确认弹窗。 - 新（batchInput.vue）：新增 `handleReAuditSingle(questionId)`，只调 `q
- *来源：工作总结*

### 3. aiCheckGroupQuestion.vue 三处 UI 优化 + 复制事件
- **解决方案**：最终方案：单行显示「全部 + 5个题号 + 更多/收起按钮」，移除之前的两段式 tagBarExpanded 切换。 - 按钮「更多」配 `arrow-down` 图标，展开后「收起」配 `arrow-up` 图标（svgIcon 全局组件，无需 import）。
- *来源：工作总结*

### 4. 严重踩坑：克隆 type.ts 时误覆盖了原版完整类型定义
- **技能点**：*问题截图**：`[plugin:vite:vue] [@vue/compiler-sfc] Unresolvable type reference or unsupported built-in utility type`，指向 `index.vue` 的 `defineProps<QuillEditorProps>()` 行。 **根因**：上一轮克隆隔
- *来源：工作总结*

### 5. QuillEditor watchQuill 覆盖 modelValue 为 D
- **技能点**：现象：题干 `f(x)=3\sin(\omega x+\phi)(\omega>0,\frac{\pi}{2}<\phi<\frac{\pi}{2})` 整公式打 data-typo-id 后，点替换整个公式被替换（期望只局部替换 `\frac{\pi}{2}<\phi<\frac{\pi}{2}` 为 `-\frac{\pi}{2}<\phi<\frac{
- *来源：工作总结*

### 6. 修复：data-formula 内原始 `<`/`>` 导致高亮/匹配不完整（f
- **技能点**：现象（同一题 3 个 typo：#0 stem `\frac{\pi}{2}<\phi<\frac{\pi}{2}`、#1 analysis `\frac{\pi}{2}<\phi<\frac{\pi}{2}`、#2 analysis `-\frac{\pi}{2}<\Phi<\frac{\pi}{2}`）： - `batchInput.vue` 只高亮 #
- *来源：工作总结*

