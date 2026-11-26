---
title: "工作台日报 · 2026-11-26"
date: 2026-11-26 16:40:06
categories: [工作日记]
tags: ["日报", "大模型", "形式化验证", "智能体", "AI辅助"]
author: Litongjin
disableNunjucks: true
---

# 工作台日报 · 2026-11-26

## 🔥 行业热点

- [OpenAI cuts GPT-6.2 usage: Astra hits limits 40% faster, Sol 25% faster](https://twitter.com/thsottiaux/status/2125784147889393806) — *Hacker News*
  - 📌 **内容**：OpenAI 对 GPT-6.2 系列用量做限制，提示模型服务进入更严格的配额与成本管理阶段。
  - 💡 **学习**：可学习多模型路由、限流降级和按任务选择模型，以降低 API 成本与中断风险。
  - 🧭 **拓展**：在业务中给不同 Agent 设置 token/请求预算，并验证超限后的 fallback 链路。
- [Show HN: An agent skill that optimizes code with falsifiable hypotheses](https://chszlab.github.io/AgenticAlgorithmEngineering/) — *Hacker News*
  - 📌 **内容**：一个 Show HN 项目展示用可证伪假设驱动的代码优化智能体，把调优过程变成可验证实验。
  - 💡 **学习**：可学习将基准测试、假设生成、对照实验和回滚机制结合到自动化性能优化中。
  - 🧭 **拓展**：选一个慢函数，让智能体提出假设并跑 A/B benchmark 验证收益。
- [Multiagent coding is the future](https://www.kalzumeus.com/2026/11/25/multiagent-coding-is-the-future/) — *Hacker News*
  - 📌 **内容**：文章认为多智能体协作编码将成为未来软件开发方式，强调分工、审查与集成。
  - 💡 **学习**：可学习如何拆分需求、分配角色、管理上下文和合并多个智能体产出。
  - 🧭 **拓展**：用两个编码智能体分别写实现与测试，再做人工 code review。
- [1Password Security Incident](https://1password.com/blog/update-on-nov-2026-security-incident) — *Hacker News*
  - 📌 **内容**：1Password 安全事件提醒开发者关注密码管理器和密钥体系的可信边界。
  - 💡 **学习**：可学习密钥托管、端到端加密、恢复流程、事件响应与最小权限设计。
  - 🧭 **拓展**：审计团队密码库、MFA 和备份恢复路径，模拟一次凭据泄露演练。
- [The matrix multiplication exponent is nine fourths [pdf]](https://github.com/openai/math/blob/main/preprints/The-matrix-multiplication-exponent-is-nine-fourths-November-24-2026/paper.pdf) — *Hacker News*
  - 📌 **内容**：新论文改进矩阵乘法渐近复杂度上限，可能影响高效数值计算的理论基础。
  - 💡 **学习**：可关注快速矩阵乘法、分块/递归算法和底层线性代数库的潜在优化方向。
  - 🧭 **拓展**：对比 BLAS/MatMul 基准，观察理论突破是否会落到实际库实现。
- [Much shorter version of GRH Lean verification from Anthropic](https://github.com/lsernau/grh-lean) — *Hacker News*
  - 📌 **内容**：Anthropic 相关的 GRH Lean 验证出现更短版本，显示 AI 辅助形式化证明正在变得实用。
  - 💡 **学习**：可学习 Lean 等定理证明器、机器可读证明压缩以及用 AI 辅助验证复杂推理。
  - 🧭 **拓展**：用一个小型数学命题写 Lean 证明，并尝试让 LLM 辅助简化证明步骤。
- [GPT-6.2 Astra is the first model to beat Googlers' code golf records](https://bjin.github.io/go-codegolf-benchmark/#leaderboard) — *Hacker News*
  - 📌 **内容**：GPT-6.2 Astra 在代码高尔夫中刷新记录，体现大模型生成极简代码的能力提升。
  - 💡 **学习**：可学习模型在竞赛编程、代码压缩和领域 DSL 上的提示策略与评测方法。
  - 🧭 **拓展**：拿几道 Code Golf 题目让人类和模型分别求解，比较可读性与字节数。
- [Slow Ubuntu security response times](https://www.openwall.com/lists/oss-security/2026/11/25/9) — *Hacker News*
  - 📌 **内容**：Ubuntu 安全响应速度受质疑，反映发行版补丁流程与漏洞暴露窗口之间的张力。
  - 💡 **学习**：可学习 CVE 跟踪、发行版安全公告、临时缓解措施和自动化补丁管理。
  - 🧭 **拓展**：为生产 Linux 建立漏洞订阅、灰度升级和回滚流程。
- [Porting Rockbox to an undocumented $9 Aliexpress 'YP3' MP3 Player with LLMs](https://jayd.ml/2026/11/14/rockbox-on-yp3.html) — *Hacker News*
  - 📌 **内容**：有人用 LLM 辅助把 Rockbox 移植到缺乏文档的廉价 MP3 播放器，展示 AI 参与嵌入式逆向。
  - 💡 **学习**：可学习用 LLM 解析硬件线索、交叉编译、固件移植和开源生态适配。
  - 🧭 **拓展**：找一款旧设备，尝试用 LLM 整理引脚/寄存器线索并完成最小启动。
- [I checked Clojure's core with Lean and found unreported bugs](https://s.conao3.com/blog/2026/e4f733) — *Hacker News*
  - 📌 **内容**：作者用 Lean 检查 Clojure 核心库并发现未报告缺陷，说明形式化工具能捕捉真实 bug。
  - 💡 **学习**：可学习属性规约、定理证明、函数式库正确性验证和从证明反推测试用例。
  - 🧭 **拓展**：选一个核心纯函数，写 Lean/属性测试规约并尝试发现边界条件问题。

## 🌟 GitHub 热门开源项目

- （本次未采集到，或 github_trending 未启用）

## 🚀 技能提升点（工作总结汇总）

### 1. Vue 双向 watch 防递归
- **技能点**：掌握 Vue 多状态双向同步的防循环能力，理解数组引用变化与 deep watch 的触发边界。
- **坑点**：折叠标志与面板尺寸数组互相 watch，写入新数组引用后 deep watch 又回写标志，即使值未变也会无限递归。
- **解决方案**：在每次写状态前做等值比较，只有变化才赋值；无变化直接 return，切断循环链路。
```text
const next = [left, 'auto', right]
if (next.every((v, i) => v === sizes.value[i])) return
sizes.value = next
if (folded.value !== nextFolded) folded.value = nextFolded
```
- **拓展**：可沉淀为 useBidirectionalSync 工具，或在复杂联动中优先改为单向数据源。
- *来源：admin-workspace-new 2026-08-11*

### 2. 点击冒泡边界控制
- **技能点**：掌握业务组件事件冒泡的边界控制，避免子交互误触发父级逻辑。
- **坑点**：CopyText 点击冒泡到题目卡片外层，导致复制时误触发解析展开/收起。
- **解决方案**：在业务组件使用处加 @click.stop，共享组件保持通用，不内置 stopPropagation。
```text
<CopyText :text="label" :copyText="id" @click.stop />
```
- **拓展**：可复用交互组件可补充事件语义说明或提供可选 stop 配置。
- *来源：admin-workspace-new 2026-08-11*

### 3. 工具栏按钮类型化注册
- **技能点**：掌握富文本工具栏配置化与类型收敛能力，让业务侧通过常量安全选用按钮。
- **坑点**：业务侧不知道可传按钮名，公式/填空等特殊按钮此前必须依赖 customButtons，配置分散易错。
- **解决方案**：导出 TOOLBAR_BUTTONS 常量与联合类型，补全默认映射并内置注册相关 handler。
```text
export const TOOLBAR_BUTTONS = {
  bold: 'bold',
  formula: 'formula'
} as const
export type ToolbarButtonName = keyof typeof TOOLBAR_BUTTONS
```
- **拓展**：可扩展为统一按钮注册中心，支持权限、图标和默认行为配置。
- *来源：admin-workspace-new 2026-08-11*

### 4. 面板状态单向化简化
- **技能点**：掌握复杂可拖拽面板的状态简化能力，用单一内部状态替代多状态双向同步。
- **坑点**：尺寸、折叠、持久化多套状态互相同步，增加递归更新和维护成本。
- **解决方案**：删除尺寸记忆与双向同步，保留拖拽状态；面板显隐改用条件渲染控制。
```text
<template v-if="isAiReviewer" #panel-2>
  <AiReviewer />
</template>
```
- **拓展**：可沉淀为可收起面板组件契约：显隐用条件渲染，尺寸用局部状态。
- *来源：admin-workspace-new 2026-08-11*

### 5. 编辑落盘与顺序校验
- **技能点**：掌握自动化编辑后的交付核验能力，避免并行编辑或会话切换导致内容丢失。
- **坑点**：同一文件的多个编辑基于同一快照并行应用会互相覆盖，会话切换后也可能发现编辑未落盘。
- **解决方案**：同一文件的多个编辑逐条顺序执行，交付前重新检查关键文件内容并复跑基础校验。
- **拓展**：可引入文件 diff、哈希或编辑日志校验，形成自动化落盘检查流程。
- *来源：admin-workspace-hr 2026-09-08*

### 6. 合并主干格式化策略
- **技能点**：掌握重构分支合并主干时降低冲突的能力，避免格式噪音放大语义冲突。
- **坑点**：用镜像格式化分支把主干也格式化后再合并，反而把单方修改变成双方修改，冲突显著放大。
- **解决方案**：直接 merge 主干，冲突处优先采纳主干版本，再统一格式化并基于主干重做语义成果。
```text
git merge master
npm run format
```
- **拓展**：可沉淀团队合并 SOP，明确格式化与语义修改分离。
- *来源：admin-workspace-hr 2026-10-14*

