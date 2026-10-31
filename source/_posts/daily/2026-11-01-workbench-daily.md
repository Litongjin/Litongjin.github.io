---
title: "工作台日报 · 2026-11-01"
date: 2026-11-01 07:02:11
categories: [工作日记]
tags: ["日报", "AI", "Web开发", "供应链安全", "反爬"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-01

## 🔥 行业热点

- [AI, Lean, and the Future of Formal Verification, with Lean Creator Leo de Moura](https://thedataexchange.media/leo-de-moura-lean-aws/) — *Hacker News*
  - 📌 **内容**：围绕 AI 与 Lean 定理证明器的结合，讨论形式化验证在 AI 辅助下可能的发展方向。
  - 💡 **学习**：了解交互式定理证明与形式化验证的用途，以及 AI 在证明搜索与自动化中扮演的角色。
  - 🧭 **拓展**：可安装 Lean 4 与 Mathlib 跟随官方教程做小练习，观察 AI 辅助证明工具的实际效果。
- [Anthropic blocks users for anything touching training larger models](https://twitter.com/jiahaoshao1/status/2116475922190922088) — *Hacker News*
  - 📌 **内容**：有用户反映因涉及训练更大模型等行为被 Anthropic 封禁，反映 AI 厂商对模型蒸馏与条款执行趋严。
  - 💡 **学习**：使用商业大模型 API 需留意服务条款，特别是用其输出训练其他模型的限制。
  - 🧭 **拓展**：阅读 Anthropic 使用政策与 API 条款，评估自家产品的合规风险。
- [The Impact of Generative AI on Critical Thinking: Self-Reported (2025)](https://dl.acm.org/doi/full/10.1145/3706598.3713778) — *Hacker News*
  - 📌 **内容**：一项自陈式研究探讨生成式 AI 使用与批判性思维变化之间的关系。
  - 💡 **学习**：关注 AI 辅助对认知的影响，思考在日常编码流程中如何保留人工审查与质疑环节。
  - 🧭 **拓展**：阅读原论文的方法与样本局限，设计小实验对比有无 AI 辅助时的判断质量。
- [I post trained a model on Demoscenes](https://datavorous.com/writing/demoscener/) — *Hacker News*
  - 📌 **内容**：作者用 demoscene 相关数据对模型进行后训练，探索特定风格与领域下的生成效果。
  - 💡 **学习**：了解后训练（微调/偏好优化）在小众领域数据上的实践思路与数据准备方式。
  - 🧭 **拓展**：用开源小模型加 LoRA 在自有数据集上复现，比较不同数据配比的效果。
- [5 years of OqtaDrive: Microdrive emulation to Sinclair Network over the internet](https://oqtadrive.org/10.other/5-years/) — *Hacker News*
  - 📌 **内容**：一个持续五年的项目，用微控制器模拟老式 Microdrive 磁带机，并让 Sinclair 设备通过互联网联网。
  - 💡 **学习**：可学习复古硬件的存储设备模拟与协议逆向思路，以及嵌入式设备联网桥接的实践。
  - 🧭 **拓展**：查看项目源码与硬件接线，用 ESP32 或树莓派尝试模拟其他老式外设。
- [Backdoor in Claude Code for VS Code extension on the Cursor Marketplace](https://phorion.io/blog/offical-claude-code-vsx-compromise/) — *Hacker News*
  - 📌 **内容**：有报告称 Cursor 市场上的 Claude Code VS Code 扩展存在后门，凸显第三方插件市场的供应链风险。
  - 💡 **学习**：安装 IDE 扩展时要注意来源与权限，学习如何审查插件包内容与更新记录。
  - 🧭 **拓展**：检查自己已装扩展的来源，必要时在沙箱或隔离环境中运行不信任插件。
- [HackerTube: YouTube built from videos recommended by HN](https://karpathy.bearblog.dev/hacker-tube/) — *Hacker News*
  - 📌 **内容**：一个把 Hacker News 上被推荐的 YouTube 视频聚合成浏览体验的网站。
  - 💡 **学习**：可学习如何抓取与聚合社区推荐数据，并做成轻量推荐型产品。
  - 🧭 **拓展**：参考其数据源，自己写一个 HN/RSS 聚合小工具练手。
- [Tell HN: Scribd Breached](https://news.ycombinator.com/item?id=50312756) — *Hacker News*
  - 📌 **内容**：用户报告文档分享平台 Scribd 发生数据泄露事件。
  - 💡 **学习**：关注泄露影响范围（账号、密码等），复习密码管理与凭据轮换实践。
  - 🧭 **拓展**：检查自己是否在该站注册，启用密码管理器与双因素认证，必要时更换密码。
- [Cloudflare Turnstile solver library (no headless browser required)](https://github.com/chris124567/cfsolve) — *Hacker News*
  - 📌 **内容**：一个无需无头浏览器即可求解 Cloudflare Turnstile 的库，说明人机验证仍存在被自动化绕过的空间。
  - 💡 **学习**：了解验证码/人机验证的实现弱点与自动化对抗手段，理解为何需要多层防护。
  - 🧭 **拓展**：在自己站点表单上测试 Turnstile 配置与限流策略，评估实际防护强度。
- [Do the math: How we took over ChatGPT accounts](https://www.parameter.ai/labs/we-found-a-gap-at-openai) — *Hacker News*
  - 📌 **内容**：分析者讲述如何利用逻辑与计算类缺陷接管 ChatGPT 账号的过程。
  - 💡 **学习**：可学习账号接管类漏洞的常见模式（业务逻辑缺陷、会话与令牌处理），以及授权边界校验的重要性。
  - 🧭 **拓展**：对自己负责的登录与会话流程做威胁建模，检查 OAuth 与令牌校验逻辑。

## 🌟 GitHub 热门开源项目

- [cathrynlavery/diagram-design（Agent Skills）](https://github.com/cathrynlavery/diagram-design) — *GitHub · HTML · +20.6k/周 · 总 63.5k star*
  - 📌 **是什么**：一套面向多种 AI 编码助手（Claude Code、Codex、Copilot、Cursor 等）的「编辑级图表设计」技能包，用自包含的 HTML + SVG 产出多种类型的架构图与数据图，刻意避开 Mermaid 的默认观感。
  - 💡 **学习点**：学习如何把设计规范（配色、留白、无阴影、版式）沉淀成 agent 可执行的 skill 指令与模板，而不是每次靠口头描述。
  - 🧭 **上手**：先读仓库里的 skill 说明文件与任一图表示例的 HTML 源码，再让 agent 按同一规范给你自己的项目画一张架构图。
- [tt-a1i/archify（Agent Skills）](https://github.com/tt-a1i/archify) — *GitHub · JavaScript · +16.9k/周 · 总 92.1k star*
  - 📌 **是什么**：一个 agent skill，能把一句话想法、方案文档或整个代码库转成可交互的架构图，适配 Claude Code、Codex 等编码 agent。
  - 💡 **学习点**：看它如何用 JS 在浏览器里渲染可交互图，以及如何把「读代码库→抽象结构→出图」拆成 agent 可逐步执行的流程。
  - 🧭 **上手**：挑一个你自己写过的小项目跑一次生成，再把输出的图与实际目录结构逐块对照，看哪里抽象错了。
- [rohitg00/ai-engineering-from-scratch](https://github.com/rohitg00/ai-engineering-from-scratch) — *GitHub · Python · +14.0k/周 · 总 76.1k star*
  - 📌 **是什么**：一套从零学 AI 工程的教程式仓库，覆盖深度学习、生成式 AI、LLM 与 agent 等主题，主张「学它、做它、交付它」。
  - 💡 **学习点**：可以按图索骥把 LLM/agent 的基础概念从前端工程师熟悉的工程视角补齐，而不是只停留在调 API。
  - 🧭 **上手**：先看目录定位到 LLM 或 agent 章节，挑其中第一个可运行的 notebook 或脚本跑通，再回头读对应讲解。
- [alibaba/open-code-review（开发工具）](https://github.com/alibaba/open-code-review) — *GitHub · Go · +6.3k/周 · 总 49.2k star*
  - 📌 **是什么**：一个混合架构的代码审查工具：用确定性流水线配合 LLM Agent 做仓库级上下文分析，输出精确到行级的评论，并内置多语言规则集。
  - 💡 **学习点**：学习「传统静态分析规则 + LLM」的分工设计，以及如何把模型输出稳定落到具体文件的具体行。
  - 🧭 **上手**：拿你自己一个前端仓库当输入跑一次 review，观察哪些问题由规则命中、哪些由 LLM 补充。
- [coreyhaines31/marketingskills（Agent Skills）](https://github.com/coreyhaines31/marketingskills) — *GitHub · JavaScript · +4.1k/周 · 总 57.7k star*
  - 📌 **是什么**：一套给 Claude Code 等 AI agent 用的营销技能集合，覆盖转化率优化、文案写作、SEO、分析与增长工程。
  - 💡 **学习点**：学习如何把非工程领域的流程（写文案、做 SEO 审计）拆成结构化的 skill 描述与可执行步骤。
  - 🧭 **上手**：挑 SEO 或文案其中一个 skill 的说明文件精读，再让它给你的开源项目写一版落地页文案试效果。

## 🚀 技能提升点（工作总结汇总）

### 1. Vue watch 双向同步死循环
- **技能点**：能识别「状态A→数组→状态B→状态A」的隐式回写环，并用值比较守卫把双向同步改造成幂等更新。
- **坑点**：一边 watch 折叠标志写 localPanelSizes（新数组引用），另一边 deep watch 数组回写标志；即使最终值未变，引用变化仍互相唤醒，直至 Maximum recursive updates exceeded。
- **解决方案**：每一步先算 next 再与当前值逐项比较，相等直接 return；数组与标志两侧都只在真正变化时赋值。
```text
const next = [leftWidth, "auto", rightWidth];
if (next[0] === cur[0] && next[2] === cur[2] && next[1] === cur[1]) return;
localPanelSizes.value = next;
```
- **拓展**：双向同步优先收敛成单一数据源 + computed，从根上避免两条 watch 互相驱动。
- *来源：admin-workspace-new / 2026-08-11*

### 2. 点击冒泡与修复层级
- **技能点**：沿事件传播链路定位误触发根因，并判断修复该落在业务调用点还是共享组件。
- **坑点**：Card 内 CopyText 的点击冒泡到外层容器的 @click，导致「复制」顺带触发题目展开/收起。
- **解决方案**：在业务组件调用处加 @click.stop 切断冒泡；共享 copyText 组件保持通用，不内建 stopPropagation、不加 event 参数。
```text
<CopyText :text="`题目编号: ${data.questionId}`" :copyText="data.questionId" @click.stop />
```
- **拓展**：通用交互组件一律不自作主张吞事件，让调用方按场景决定是否阻止冒泡。
- *来源：admin-workspace-new / 2026-08-11*

### 3. 共享组件能力边界
- **技能点**：扩展通用组件时保留其可选能力、由调用方决定是否启用，同时用插槽透传满足定制 UI。
- **坑点**：把持久化、折叠标志双向同步、hidden 字符串等业务语义塞进面板调用方，需求一精简就要从多处回删，还易留残留引用。
- **解决方案**：组件侧 storageKey 为空时自动跳过 saveStorage、新增 #start-collapsible 插槽透传，隐藏官方图标；调用方只传自己需要的配置，显隐改用 v-if 控制。
```text
const saveStorage = () => {
  if (!props.storageKey) return;
  localStorage.setItem(props.storageKey, JSON.stringify(sizes.value));
};
```
- **拓展**：新增能力默认「不传即无副作用」，并配一眼可用的默认值，减少调用方心智负担。
- *来源：admin-workspace-new / 2026-08-10、2026-08-11*

### 4. 白名单常量与类型导出
- **技能点**：用常量对象 + 字面量联合类型把「字符串魔法值」升级为可发现、可校验的组件 API。
- **坑点**：业务组件不知道 :toolbar-order 支持哪些按钮名，公式/填空还必须额外配 customButtons 才能用，能力被隐式限制。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与 ToolbarButtonName 类型，补全默认按钮映射、内置 handler 无条件注册，调用方用 B.xxx 声明。
```text
export const TOOLBAR_BUTTONS = { undo: 'undo', redo: 'redo', fillInInsert: 'fillInInsert', formula: 'formula' } as const;
export type ToolbarButtonName = typeof TOOLBAR_BUTTONS[keyof typeof TOOLBAR_BUTTONS];
// toolbarOrder?: ToolbarButtonName | string
```
- **拓展**：同类枚举型配置（列 key、事件名、插槽名）统一走「常量 + 联合类型」，再考虑对外发布类型声明。
- *来源：admin-workspace-new / 2026-08-11*

### 5. 编辑落盘与改动核对
- **技能点**：对「看似已改」的结论保持怀疑，改完回读磁盘/复核引用，确认改动真正生效。
- **坑点**：会话切换后工作区被重置，前期对 toolbarConfig.ts、detailComment.vue 的 edit 实际未落盘，磁盘仍是旧版，差点误判为已完成。
- **解决方案**：收尾时重新读取目标文件比对内容，重新正确应用改动，并用 grep 确认被删符号 0 处残留引用。
- **拓展**：批量改动后固定跑一遍「符号残留 grep + lint 增量」自检，作为交付前的机械步骤。
- *来源：admin-workspace-new / 2026-08-11*

### 6. 主干合并策略
- **技能点**：能在「格式化一致性」与「合并冲突量」之间做实测取舍，选择冲突最小的合并路径。
- **坑点**：为对齐格式先做全量 prettier 的镜像分支再 merge，反而放大冲突（117 文件/414 冲突块 vs 直接 merge 的 51/118），因为合并 base 是未格式化版本。
- **解决方案**：直接 git merge 主干 → 冲突处整体采纳主干版 → 统一跑一次格式化 → 再按清单 grep 驱动地在主干基线上重做语义改动。
```text
git merge master
# 冲突处整体采纳主干版本
npm run format
grep -rn "<我方新增符号>" src/  # 按清单逐项重做语义改动
```
- **拓展**：把该流程写进合并实施方案文档，作为后续重构分支回归主干的固定 SOP。
- *来源：admin-workspace-hr-talent / 2026-10-14*

