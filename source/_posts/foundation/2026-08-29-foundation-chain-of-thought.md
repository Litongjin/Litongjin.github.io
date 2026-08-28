---
title: "每日基础技术总结 · 2026-08-29 · Chain-of-Thought（思维链）基础"
date: 2026-08-29 06:55:34
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-29 · Chain-of-Thought（思维链）基础

## 📚 今日主题

> **Chain-of-Thought（思维链）基础**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Chain-of-Thought（思维链）是一种提示工程技术，通过在生成最终答案前让模型先输出一段中间推理步骤（token 序列），改变自回归解码的条件概率分布。其本质是在输入与输出之间插入一组可观测的隐变量，将一步到位的概率估计 P(answer|question) 分解为 P(step1|q)·P(step2|q,step1)···P(answer|q,step1..n)。解决的核心问题是：Transformer 在单次前向传播中隐式完成复杂推理时，中间状态受层深与上下文窗口限制，无法可靠地长距离传递；思维链把计算外化为文本 token，使后续每一步都能显式读取已生成的中间结果，相当于给模型提供了一张外部草稿纸。在 AI 体系中，它属于 inference-time scaling 的基础方法，是 prompting engineering、Agent 规划、过程监督等技术的底层前提。专业工程师必须掌握它，因为它决定了对 LLM 输出信任边界、调试手段（是否可以通过查看中间链定位错误）以及系统设计中“是否要提示模型逐步思考”的关键判断。

### 2. 底层原理剖析
从机制上看，LLM 是 token 级自回归生成：P(w_1..w_T)=∏P(w_t|w_<t)。无思维链时，模型需要从问题直接回归到答案，所有中间计算只能在 attention 与 FFN 的隐状态中隐式表示，受限于层深和宽度。有思维链时，中间 token 被显式生成并加入上下文，每个后续 token 的预测条件变成 P(w_t | question, z_1..z_{t-1})，其中 z 是思维链的文本。由于每一步只负责一个小的写出步骤，条件熵远低于一步直接预测答案，因此模型更容易输出正确结构。这与前端工程中的管线重构同构：把单个复杂的 .map().filter().reduce() 链式调用里的中间值通过变量暴露，而不是在单个闭包里隐式累积；也类似 Redux 中间件逐层处理 action，每个中间件能看到前一个产生的状态。区别于 Java 接口和 TS 接口：Java 接口是运行时类型约束，TS 接口是编译期结构约束；普通生成是“隐式满足输出契约”，思维链则是“显式写出每一步如何满足契约”——它把推理协议暴露在文本层，而普通生成把协议埋在权重里。但需要注意，思维链文本不是编译器中间表示，它没有受控的语义；它只是统计上倾向于生成与训练语料中推理模式一致的文本。

### 3. 基础代码与实战验证
```text
# 伪代码：思维链的本质是在上下文中追加中间 token

def generate(model, context, stop):
    out = ''
    while not context.endswith(stop):
        token = argmax(model.predict(context))  # 根据完整上下文预测下一个 token
        context += token                        # 关键：将中间 token 显式写回上下文
        out += token
    return out

# 标准模式：一步从问题映射到答案
answer = generate(model, 'Q:17*23=? A:', stop='\n')

# 思维链模式：先生成中间步骤，再基于完整上下文生成答案
ctx = 'Q:17*23=? A: Let\'s think step by step.'
reasoning = generate(model, ctx, stop='Answer:')
answer_cot = generate(model, ctx + reasoning, stop='\n')

# 概率分解：
# 无 CoT: P(answer | Q) —— 一步高维映射
# 有 CoT: P(reasoning | Q) * P(answer | Q, reasoning) —— 级联低条件熵预测
```

### 4. 常见误区与进阶思考
误区一：把思维链当作模型真实的推理轨迹。实际上语言模型没有独立的推理引擎，思维链文本只是符合训练数据分布的一种续写，可能包含看似合理但错误的逻辑；它在统计意义上提供的是可读的中间假设，而不是可验证的内部计算过程。
误区二：认为思维链是万能无条件增强。它对需要多步归纳/演绎的任务有效，对简单任务可能引入冗余 token 与错误漂移；效果依赖模型规模、数据和提示措辞，小模型往往无法受益。
深入思考题：既然思维链只是自然语言的继续生成，为什么在数学题上能显著提升答案正确率？请从自回归条件概率分解与隐变量显式化的角度解释其机制，并构造一个 CoT 会失效的反例（例如：训练分布中不存在该推理模式，或中间 token 本身无法被可靠生成）。
