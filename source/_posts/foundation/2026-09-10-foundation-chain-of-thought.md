---
title: "每日基础技术总结 · 2026-09-10 · Chain-of-Thought（思维链）基础"
date: 2026-09-10 07:01:19
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-10 · Chain-of-Thought（思维链）基础

## 📚 今日主题

> **Chain-of-Thought（思维链）基础**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Chain-of-Thought（CoT）是一种提示工程技术，其本质是在输入中显式要求语言模型在输出最终答案之前，生成一系列中间推理步骤。它解决的问题是：在自回归语言模型中，模型被训练为逐 token 预测下一个 token，缺少显式的计算中间状态。对于需要多步组合、代数运算或逻辑推导的任务，直接生成答案往往导致一步到位的概率极低甚至不可能；CoT 将隐式推理过程外部化为显式中间序列，使每一步都在模型上下文窗口中成为后续步骤的条件，从而将困难问题的求解分解为多个条件概率的连乘。机制上，它利用 transformer 的自注意力与上下文存储能力，让中间步骤参与对最终答案的注意力计算，本质是引导模型将内部计算过程外显化，而非引入新的参数或推理引擎。在整个 AI 体系中，它位于 LLM 对齐与能力激活层，属于没有改变模型权重却改变任务可解性的上下文工程。专业工程师必须掌握它，因为它是 Agent、复杂任务规划、工具调用和可靠输出的基石；不理解 CoT 就无法理解为何简单提示与结构化提示之间存在巨大的能力鸿沟，也无法诊断 LLM 应用中的系统性失败。

### 2. 底层原理剖析
底层机制可抽象为：给定输入 x，标准语言模型直接建模 P(answer | x)，而 CoT 建模 P(thought_1,...,thought_n, answer | x)，且由于自回归解码的性质，整个序列的概率为 ∏ P(token_i | x, tokens_<i)。中间 thought 序列充当了外部记忆与计算寄存器，把一次复杂映射变成多次简单映射的链。从信息论角度看，CoT 并未增加可用的模型容量，而是改变了 decoder 的生成路径：每一步的生成都只依赖于当前上下文窗口的注意力权重，因此中间步骤是被强制写入上下文的‘中间变量’，让模型可以从这些中间变量中重新读取并继续计算。这与前端工程中的接口不同：TypeScript 的接口是编译期的类型契约，在运行时被擦除，不参与任何实际逻辑；而 CoT 的‘接口’是可执行的 token 序列，它本身参与生成，并动态影响后续输出。更相似的类比是 React 的 render 函数中显式拆解 state 转换步骤：将复杂 UI 状态变化拆为多个中间变量，每一步都驱动下一次渲染；CoT 的中间 token 就是模型空间中的‘中间变量’。另一个底层要点是：CoT 有效的前提是模型在预训练阶段已经隐式学习到了大量逐步推导的数据分布；CoT 只是激活这种既有能力，而非凭空创造。伪代码可视为：若任务需要多步推理：(1) 把问题 Q 包装成“Let's think step by step”等显式指令；(2) 让模型生成第 1 个中间结论 c1；(3) 将 Q + c1 拼接作为输入，继续生成 c2；(4) 重复直到生成最终答案 A。在数学上，每一步 c_i 都是文本，模型对 c_i 的生成同时受 Q 和之前 c_j(0<j<i) 的约束，从而形成一条可被注意力机制回溯的计算轨迹。

### 3. 基础代码与实战验证
```text
由于 CoT 是提示层技术，不需要模型训练，验证手段是调用 LLM API。核心伪代码/步骤（不依赖框架）：

def chain_of_thought(query, model, tokenizer, max_steps=5):
    # 1. 构造显式引导提示，强制模型输出中间推理步骤
    prompt = query + "\n\nLet's think step by step before answering."

    # 2. 自回归解码：每次只生成一个 token，直到遇到停止标记
    outputs = []
    current_input = prompt
    while True:
        # 将当前全部文本编码成 token ID 序列
        input_ids = tokenizer.encode(current_input)
        # 前向传播得到下一个 token 的概率分布
        logits = model(input_ids).logits[:, -1, :]
        # 贪心采样取概率最大的 token id
        next_id = argmax(logits)
        next_token = tokenizer.decode(next_id)
        outputs.append(next_token)
        # 将新生成的 token 拼入上下文，作为下一步的条件
        current_input = current_input + next_token
        # 回合终止条件：遇到换行+答案分隔符，或输出长度超限
        if next_token in ['\n', '<|endoftext|>'] or len(outputs) > max_steps:
            break
    # 3. 从生成文本中解构出中间链和最终答案
    reasoning, answer = extract_final_answer(current_input)
    # 返回值：完整 CoT 轨迹 + 最终答案
    return {'chain': outputs, 'answer': answer}

# 对照实验（验证本质）：移除 CoT 指令后，direct_answer = model.generate(query)，比较准确率。
# 关键点：伪代码中 while 循环的每次拼接 current_input，就是 CoT 机制的核心——将先前中间步骤反馈进注意力矩阵，使后续 token 的生成依赖它们。
# 在实际工程中，该循环由 decoder 内部实现，但提示词影响的是采样路径；理解这个循环即理解 CoT 的原理。
```

### 4. 常见误区与进阶思考
误区一：认为 CoT 是让模型变得‘会推理’，即模型学会了逻辑规则。实际上 CoT 没有改变任何权重，它只是将模型内部原本难以显式采样的推理轨迹通过提示引导到文本空间；模型仍然在做概率化 token 预测，并不保证每一步逻辑自洽。若领域任务在预训练中极少出现逐步推导语料，CoT 可能无效甚至产生更长但不正确的输出。误区二：认为‘Let's think step by step’就是 CoT 的万能钥匙，忽略任务复杂度与格式设计。底层要求是中间步骤必须能被自注意力有效关联；如果中间步骤过长导致超出有效上下文窗口或注意力衰减，CoT 的收益会被抵消。
思考题：在寄存器有限的硬件上执行一个 8 位状态机，如果把状态转移的每一步都打印到串口（CoT），然后在下一轮读取串口内容继续运行，最终结果与不打印直接内部跳转运行是否可能不同？若优化器在训练时将串口打印设置为不可学习的死代码，那么 CoT 还有效吗？请结合 token 生成的条件概率链解释——这考察的是对‘CoT 本质是在模型权重之外建立外部可读状态’的理解。
