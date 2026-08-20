---
title: "每日基础技术总结 · 2026-08-19 · Zero-shot / Few-shot 原理"
date: 2026-08-19 18:41:56
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-19 · Zero-shot / Few-shot 原理

> 自动生成于 2026-08-19 18:41 · 个人工作台 Agent

## 📚 今日主题

> **Zero-shot / Few-shot 原理**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Zero-shot 与 Few-shot 是大型语言模型（LLM）在推理阶段通过输入提示（prompt）而非参数更新来适配新任务的机制。Zero-shot 指在输入中仅包含任务指令或问题描述，不提供任何示例，模型依赖预训练阶段习得的语言知识与指令理解能力直接输出结果。Few-shot 则在输入中提供 k 个（通常 1~数十个）输入-输出示例作为条件上下文（in-context examples），模型根据这些示例推断出映射模式并作用于新输入。二者本质上是将任务形式化描述为条件概率 P(output | instruction, examples, input)，其中 examples 可为空。它们解决的核心问题是：如何在不对模型参数进行梯度更新（即不训练、不微调）的情况下，利用已有预训练知识完成新任务。其底层机制依赖 Transformer 的自注意力层对上下文 token 序列进行模式匹配与类比推理，以及预训练语料中蕴含的任务分布先验。在整个 AI 体系中，Zero-shot/Few-shot 属于『上下文学习』（In-Context Learning）范畴，是介于传统监督学习和参数高效微调之间的轻量级适配手段。专业工程师必须掌握它，因为这是 LLM 时代实现任务泛化的最小开销路径，也是设计 Agent、RAG、工具调用等复杂系统的核心前提，理解其原理才能正确评估模型能力边界并设计有效的提示策略。

### 2. 底层原理剖析
底层机制可从三个层面剖析：
1. 序列建模层面：LLM 本质是一个在 token 序列上自回归的条件概率分布模型。给定上下文 C = [instruction; example_1; ...; example_k; input]，模型按位置逐 token 生成 P(token_i | C, token_<i)。Few-shot 示例与输入被编码为同一连续的 token 序列，在 Transformer 的自注意力层中，每个 token 的表示会通过 QKV 交互聚合所有其他 token 的信息，因此输入中的示例模式会通过注意力权重影响后续生成概率。Zero-shot 则是 C 中仅有指令和输入，模型必须从参数记忆中直接检索与指令匹配的隐式任务模式。
2. 任务表示层面：预训练阶段，模型在互联网级文本上学习到大量『任务描述→任务执行』的联合分布。例如代码库中常见『# 排序列表
# 输入: [3,1,2] # 输出: [1,2,3]』这类模式。Zero-shot 依赖模型将自然语言指令映射为已学习过的任务表征；Few-shot 通过显式示例激活更具体的映射路径，相当于在隐空间中约束任务方向。这与前端中『接口（Interface）』的本质区别在于：TS/Java 接口是编译期的静态契约，编译器根据类型结构强制约束实现；而 Few-shot 示例是运行期的动态条件，模型根据上下文概率性地『推断』任务，没有编译期校验，也没有保证正确性的契约。若将 Few-shot 比作类型推导，则它是基于统计的软推导，而非基于类型规则的硬推导。
3. 优化视角：Few-shot 的 k 个示例实际上充当了经验风险最小化中的『训练样本』，但损失函数并非最小化输出误差，而是最大化给定示例条件下的生成似然。模型参数不变，示例仅改变隐层状态的条件分布。其效果受示例的顺序、多样性、与输入分布的距离影响，这类似于前端中 CSS 选择器的优先级——但选择器是确定性规则，而 Few-shot 的影响是非线性且概率性的。
伪代码表示：
for each task T:
  build prompt = instruction + examples[0:k] + input
  logits = LLM.forward(prompt)  # 参数冻结，前向传播
  output = sample(logits)
  # 无反向传播，无参数更新
关键点：示例数量 k 通常在 1~16 之间效果最佳，过大的 k 可能超出模型的有效上下文窗口或引入噪声；且示例与测试输入在语义空间上的对齐程度比数量更重要。

### 3. 基础代码与实战验证
```text
以下用 PyTorch + HuggingFace Transformers 演示 Few-shot 与 Zero-shot 的最小实现，不含任何训练循环。

from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "microsoft/phi-2"  # 任意因果语言模型
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)

# 构建 Zero-shot 提示：仅指令 + 输入
def zero_shot_prompt(task: str, input_text: str) -> str:
    return f"{task}\n输入: {input_text}\n输出:"

# 构建 Few-shot 提示：指令 + k 个示例 + 输入
def few_shot_prompt(task: str, examples: list[tuple[str, str]], input_text: str) -> str:
    prompt = task + "\n"
    for ex_input, ex_output in examples:
        prompt += f"输入: {ex_input}\n输出: {ex_output}\n"
    prompt += f"输入: {input_text}\n输出:"
    return prompt

# 生成函数：前向传播，仅采样，不更新梯度
def generate(prompt: str, max_new_tokens: int = 16) -> str:
    inputs = tokenizer(prompt, return_tensors="pt")
    # model.eval() 关闭 dropout 等训练专用层，但参数不变
    with torch.no_grad():
        outputs = model.generate(**inputs, max_new_tokens=max_new_tokens)
    return tokenizer.decode(outputs[0][inputs["input_ids"].shape[1]:], skip_special_tokens=True)

# 任务：判断数字的奇偶性（模型可能已见过，但演示机制）
task = "判断一个整数是奇数还是偶数。"
input_1 = "5"

# Zero-shot 调用
print(generate(zero_shot_prompt(task, input_1)))  # 期望: 奇数，但依赖预训练分布

# Few-shot：提供两个示例，让模型通过上下文类比
examples = [("2", "偶数"), ("7", "奇数")]
input_2 = "4"
print(generate(few_shot_prompt(task, examples, input_2)))  # 期望: 偶数

# 关键注释：
# - prompt 字符串被 tokenizer 编码为 token id 序列，作为模型输入。
# - model.generate 内部循环执行 self-attention 和 FFN，每一步预测下一个 token。
# - 示例的作用是改变 self-attention 中 query/key 的匹配权重，使『输入-输出』模式被激活。
# - 没有反向传播，模型参数保持为预训练权重，这就是 in-context learning 的底层本质。
```

### 4. 常见误区与进阶思考
误区 1：认为 Few-shot 是『在模型内部进行学习/微调』。实际上参数完全冻结，示例只影响输入上下文，模型权重不变。这与前端中『在运行时修改原型链』类似——原型链修改会影响后续查找，但不会改变构造函数的源代码；而 Few-shot 连原型链都不修改，只是改变了当前调用时的参数。若混淆，则无法理解为什么示例顺序会显著影响结果，也无法解释为什么删除示例后模型立即恢复原行为。
误区 2：认为示例数量越多效果越好，或示例质量无关紧要。事实是，LLM 的上下文窗口有限，且注意力机制对远端 token 的依赖呈衰减趋势；过多示例会稀释注意力权重，且若示例分布与目标输入分布不一致，会引入偏差。这与前端中『一次性引入过多全局 CSS 类』类似——样式冲突和优先级覆盖会导致预期外的渲染，示例过多也会导致模型输出偏向示例的共性而非任务本质。
思考题：假设你有一个二分类任务（正面/负面情感），预训练模型在 Zero-shot 下准确率为 80%。你构造了 4 个 Few-shot 示例，其中 3 个示例的标签顺序是『负面→正面』，1 个是『正面→负面』，发现模型准确率下降到 70%。请从 Transformer 注意力机制的角度解释：为什么示例的顺序和标签一致性会影响结果？如果让你设计一种固定示例顺序的算法，使得模型对示例顺序不敏感，你会怎么做？这能帮你验证自己是否真正理解 Few-shot 的底层机制。
