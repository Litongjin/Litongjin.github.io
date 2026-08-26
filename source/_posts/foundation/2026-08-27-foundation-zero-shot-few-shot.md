---
title: "每日基础技术总结 · 2026-08-27 · Zero-shot / Few-shot 原理"
date: 2026-08-27 06:55:58
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-27 · Zero-shot / Few-shot 原理

## 📚 今日主题

> **Zero-shot / Few-shot 原理**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Zero-shot / Few-shot 是大型语言模型（LLM）在推理阶段通过提示（Prompt）中的示例或任务描述来引导模型完成未显式训练过的任务的能力。Zero-shot 指不提供任何输入-输出示例，仅依赖模型预训练阶段习得的语言规律与指令理解能力，直接根据任务描述生成输出；Few-shot 指在 Prompt 中提供少量（通常 2~8 个）输入-输出示例，利用模型对上下文模式的隐式统计拟合，使其在推理时对齐示例所展示的映射关系。其本质不是参数更新，而是通过条件概率分布的条件化（Conditioning）来激活预训练权重中已存在的知识路径。它解决的问题是：在无法或不便微调模型时，如何以极低成本让模型泛化到新任务。机制上是将任务描述和示例编码为自然语言序列，模型自回归解码时，每个 token 的条件概率 P(token|context) 被上下文中的模式所偏置。在计算机/AI 体系中，它属于 In-Context Learning（上下文学习）范畴，是预训练-提示范式下的核心推理技术，介于传统监督微调与参数高效微调之间。专业工程师必须掌握它，因为它是 Agent、RAG、自动化流程中控制模型行为的首要手段，直接影响任务成功率、成本与延迟，且是理解模型能力边界和幻觉来源的基础。

### 2. 底层原理剖析
底层机制可拆解为三个层次：1) 预训练阶段的分布先验：模型在海量文本上通过自回归目标学习 token 序列的联合概率 P(x1,...,xn)。该分布中已隐式编码了大量任务模式（如问答、翻译、代码补全）。2) 推理阶段的上下文条件化：给定 Prompt 前缀 C（含任务描述和示例），模型计算 P(y|x, C)。由于 Transformer 的注意力机制允许 C 中的每个 token 与后续 token 交互，示例中的输入-输出映射会被编码为 attention pattern，从而在解码时改变 softmax 输出的概率分布，使模型倾向于生成与示例结构相似的输出。3) 模式匹配而非学习新规则：Few-shot 不更新任何权重，模型只是在高维语义空间中寻找与示例最相似的隐式向量轨迹。示例的作用是定义一个局部“流形”，约束解码路径。更精确地说，可视为在贝叶斯框架下，用示例作为先验证据，调整后验预测分布。伪代码描述：

def in_context_predict(model, task_desc, examples, query):
    # examples: [(input_i, output_i)]
    prompt = task_desc + "\n"
    for inp, out in examples:
        prompt += f"Input: {inp}\nOutput: {out}\n"
    prompt += f"Input: {query}\nOutput:"
    # 自回归解码，每一步取 P(next_token | prompt + already_generated)
    return model.generate(prompt, max_new_tokens=K)

对比前端概念：Java 接口与 TypeScript 接口本质区别在于 Java 接口是编译期契约，约束实现类的结构，强制类型检查；TS 接口是结构类型（Structural Typing），只在编译期存在，运行时被擦除。Zero-shot/Few-shot 与此的深层异同：相同点是二者都通过“声明”来约束后续行为——接口约束代码结构，Prompt 约束生成分布；但接口是硬约束（编译失败或运行时报错），而 Few-shot 是软约束（仅改变概率，不保证遵从）。更进一步，Java 接口对应“参数更新”的强约束（类实现接口则必须提供方法），Few-shot 对应“上下文注入”的弱约束（模型可忽略示例）。TS 的结构化类型与 Few-shot 的模式匹配有相似性：TS 只要结构兼容即通过类型检查，模型只要语义模式相似即可泛化；但 TS 是确定性的，模型是概率性的。另一个对比：前端中“设计模式”是复用代码结构，Few-shot 是复用“任务结构”，但前者是显式规则，后者是隐式分布偏移。

### 3. 基础代码与实战验证
```text
以下用最小化 PyTorch + HuggingFace Transformers 代码验证 Zero-shot 与 Few-shot 对生成概率的影响（不依赖复杂框架，仅需 transformers 和 torch）：

from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

# 加载一个小型因果语言模型（例如 GPT-2），展示条件概率变化
tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")
model.eval()

# 任务：情感分类（正面/负面）
# Zero-shot 提示：仅描述任务
task_desc = "Classify sentiment as positive or negative."
zero_shot_prompt = task_desc + "\nText: I love this movie.\nSentiment:"

# Few-shot 提示：加上两个示例（映射模式）
few_shot_prompt = (
    task_desc + "\n"
    "Text: The food was awful.\nSentiment: negative\n"
    "Text: The service was great.\nSentiment: positive\n"
    "Text: I love this movie.\nSentiment:"
)

def compute_next_token_probs(prompt: str, target_tokens: list[str]):
    # 编码输入，保留 attention mask
    inputs = tokenizer(prompt, return_tensors="pt")
    with torch.no_grad():
        # 前向传播，得到 logits
        logits = model(**inputs).logits  # shape: [batch, seq_len, vocab]
    # 取最后一个位置的 logits，即下一个 token 的概率分布
    last_logits = logits[0, -1, :]
    probs = torch.softmax(last_logits, dim=-1)
    result = {}
    for t in target_tokens:
        token_id = tokenizer.encode(t, add_special_tokens=False)[0]
        result[t] = probs[token_id].item()
    return result

# 比较在“positive”和“negative”两个词上的概率
print("Zero-shot:", compute_next_token_probs(zero_shot_prompt, [" positive", " negative"]))
print("Few-shot:", compute_next_token_probs(few_shot_prompt, [" positive", " negative"]))

# 预期输出：Few-shot 中“ positive”概率显著高于 Zero-shot，且“ negative”概率下降。
# 底层原理：注意力机制使示例中的“positive”与“negative”标签信息通过 attention 路径影响最后位置的表征，
# 从而改变了 softmax 输出的分布。注意：tokenizer 会在词前加空格，需在目标词中包含前导空格。

# 若无法运行，用伪代码描述：
# 1. 构造含任务描述与若干 (input, output) 对的字符串。
# 2. 将字符串 tokenize 后输入 transformer 模型。
# 3. 取最后一个 token 位置的 logits，softmax 后得到整个词表的概率分布。
# 4. 对比有无示例时目标答案 token 的概率变化。
# 5. 该变化即为 In-Context Learning 的数值体现。
```

### 4. 常见误区与进阶思考
误区1：将 Few-shot 理解为“微调”或“模型记住了示例”。实际上模型参数完全不变，示例只影响推理时的条件分布。若示例与真实任务分布不一致，模型可能被误导（如示例中标签顺序影响输出），且示例数量增加并不一定单调提升效果，有时 1-shot 反而优于 5-shot（因模型对长上下文的注意力分散）。
误区2：认为 Zero-shot 是“模型完全没有见过该任务”。Zero-shot 只是推理时不提供示例，但预训练数据中可能已包含大量类似任务，模型只是通过指令描述激活既有能力。因此 Zero-shot 的成败高度依赖任务描述的语言形式（措辞、格式、关键词），并非“无中生有”。
思考题：给定一个二分类任务，如果你在 Few-shot 示例中全部使用“正样本”作为输入，但输出标签随机为“yes”或“no”，且正负样本比例不变，模型最终在测试时输出“yes”的概率会如何变化？请用条件概率和注意力机制解释：模型是否会忽略标签的语义，而仅学习输入与标签的“共现”模式？这反映了 In-Context Learning 的哪种本质属性？
