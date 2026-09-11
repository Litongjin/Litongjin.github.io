---
title: "每日基础技术总结 · 2026-09-12 · Zero-shot / Few-shot 原理"
date: 2026-09-12 07:02:11
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-12 · Zero-shot / Few-shot 原理

## 📚 今日主题

> **Zero-shot / Few-shot 原理**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Zero-shot与Few-shot是LLM通过输入-输出示例的提供量来引导模型执行新任务的技术，本质上是利用模型在预训练阶段习得的语言分布知识，在推理时通过上下文学习（In-Context Learning）动态调整条件概率分布，使模型无需权重更新即可适应特定任务。Zero-shot指不提供任何示例，仅依赖任务指令（如自然语言描述）让模型直接生成输出；Few-shot则提供少量（通常1-64个）完整的输入-输出对作为示范，让模型从这些示例中隐式推断映射规则。其解决的核心问题是『如何在没有任务特定训练数据或无法负担微调成本时，快速让模型执行新任务』。机制上，它并不改变模型参数，而是将示例作为前缀拼接在输入序列中，利用Transformer的自注意力机制让模型在生成每个token时，条件于此前所有上下文（包括示例），从而将示例中的模式统计性地反映到后续生成中。在AI体系中，它属于Prompt Engineering与LLM对齐的核心基础，也是Agent系统、结构化输出、工具调用等上层能力的前提。专业工程师必须掌握它，因为它是成本最低、迭代最快的模型行为控制手段，且理解其原理才能正确评估模型能力边界、设计提示词策略，避免在无法微调的场景中盲目依赖示例或指令。

### 2. 底层原理剖析
底层原理可拆解为三点：
1. 条件概率建模：LLM本质是语言模型，其训练目标是最大化 P(y|x) = Π P(y_t | y_<t, x) 的log似然。在Zero/Few-shot场景下，输入序列实际变为 [指令; 示例1; 示例2; ...; 查询输入]，整个序列作为条件上下文，模型输出 y 的条件概率为 P(y | [指令; 示例; 查询])。由于Transformer的attention机制（尤其是causal attention），每个位置的表示都能直接或间接聚合所有前序token的信息，因此示例中的规律会通过attention权重影响最终输出分布。
2. 隐式规则提取：Few-shot中，模型并非显式编码『示例中的映射规则』，而是在自回归生成过程中，通过内部注意力头识别示例间共有的模式（如输入格式、输出风格、对应关系），并将这种模式作为先验偏置。研究表明，这类似于一种‘隐式贝叶斯推断’——模型在概率空间中对可能的映射函数进行后验采样，示例数量越多，后验分布越集中，输出越稳定。但该推断并不保证正确，因为模型可能被示例的表面特征（如长度、词汇）误导，而非真正理解抽象规则。
3. 与前端概念的对比：可以把LLM比作一个巨大的‘函数重载分发器’，Zero-shot类似于使用TypeScript接口（interface）定义入参类型和返回值类型——只描述契约，不提供具体实现，但要求编译器能自动生成实现（依赖类型推断）。Few-shot则类似于Java中的接口实现类——你提供几个具体的类实例（示例），让框架通过反射机制推断接口的通用实现模式，但这里的推断是统计性的而非确定性的。更贴切的对比是：前端中‘组件复用’的prop约定——你传入不同数量的示例（props）来控制组件的渲染行为，但组件内部逻辑（模型参数）不变，只是props的值改变了输出模式。Zero-shot相当于只传入一个render函数描述需求，Few-shot相当于传入几个示例片段来暗示样式。关键差异在于：前端接口/TS类型是编译期静态检查，而LLM的上下文学习是运行时动态推断，没有类型安全，也没有编译器报错，只能通过输出质量来评估。
4. 伪代码流程：
   def infer(model, instruction, examples, query, shot):
       if shot == 0:
           prompt = instruction + '\n' + query
       else:
           prompt = instruction + '\n'
           for (input, output) in examples[:shot]:
               prompt += f'{input} => {output}\n'  # 将示例拼接为统一格式
           prompt += query
       output_tokens = model.generate(prompt, max_new_tokens=...)
       return decode(output_tokens)
   实际生成时，模型对每个输出token计算概率分布，并选择最高概率token（或采样），整个过程无梯度计算，无参数更新。

### 3. 基础代码与实战验证
```text
以下为极简的验证代码（使用HuggingFace Transformers，但逻辑不依赖框架）：

from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

model = AutoModelForCausalLM.from_pretrained('gpt2')
tokenizer = AutoTokenizer.from_pretrained('gpt2')

# 构造Zero-shot场景：仅给指令，不给示例
def zero_shot(prompt):
    # 将用户提示转换为模型输入，并生成token保持为提示部分
    input_ids = tokenizer.encode(prompt, return_tensors='pt')
    output = model.generate(
        input_ids,
        max_new_tokens=20,       # 最多生成20个新token
        do_sample=False,         # 贪心解码，便于复现
        pad_token_id=tokenizer.eos_token_id
    )
    return tokenizer.decode(output[0][input_ids.shape[1]:])  # 仅解码新生成部分

# 构造Few-shot场景：在输入序列前拼接两个示例，让模型观察映射规律
def few_shot(instruction, examples, query):
    # examples: [(input1, output1), (input2, output2)]
    prompt_parts = [instruction]
    for inp, out in examples:
        # 将每个示例格式化为“输入 => 输出”并换行，使自注意力能关联对应关系
        prompt_parts.append(f"{inp} => {out}")
    prompt_parts.append(query)
    prompt = '\n'.join(prompt_parts)  # 拼接为单个序列，模型将此序列视为一个整体条件上下文
    input_ids = tokenizer.encode(prompt, return_tensors='pt')
    output = model.generate(
        input_ids,
        max_new_tokens=20,
        do_sample=False,
        pad_token_id=tokenizer.eos_token_id
    )
    return tokenizer.decode(output[0][input_ids.shape[1]:])

# 验证：对比Zero-shot与Few-shot在“将英文单词转为大写”任务上的表现
instruction = "Turn the following word to uppercase:"
print(zero_shot(instruction + "\napple"))  # 很可能输出错误或单词本身（GPT-2未微调指令跟随）

# 提供两个示例，引导模型理解输出格式
examples = [("cat", "CAT"), ("dog", "DOG")]
print(few_shot(instruction, examples, "apple"))  # 因为示例中已出现“输入=>大写输出”模式，模型更可能输出“APPLE”

# 底层机制：模型在每个生成步，将已经生成的所有token作为key的query，通过自注意力计算当前token的上下文表示，
# 因此示例中的映射对会直接影响最终softmax层对每个候选token的得分。
```

### 4. 常见误区与进阶思考
常见误区1：认为Few-shot是让模型“记住”示例或进行梯度更新。实际没有权重变化，示例只是作为前缀token参与前向计算。若任务复杂且示例数量超过上下文窗口，模型可能因注意力分散而忽略早期示例，或仅复制表面格式而误解语义。
常见误区2：将Few-shot的示例数量等同于性能单调递增。实际存在“超越临界点”现象——当示例超过一定数量时，模型可能陷入模式过拟合，性能反而下降。因为上下文长度增加导致注意力权重被稀释，且冗余示例可能引入噪声，使隐式推断的置信度降低。
思考题：在Few-shot中，如果将示例的顺序完全随机打乱（不改变示例对本身的内容），模型输出表现可能剧烈波动，但前端中接口实现类的顺序不会影响行为。请从Transformer自注意力机制和隐式概率推断的角度解释：为什么示例顺序会影响LLM的上下文学习效果？你能设计一个实验来验证这种顺序敏感性吗？
