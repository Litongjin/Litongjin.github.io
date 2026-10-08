---
title: "每日基础技术总结 · 2026-10-08 · Function Calling 与工具调用规划"
date: 2026-10-08 07:04:50
categories: [技术分享]
tags: ["技术分享", "AI / LLM 工程实战"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-08 · Function Calling 与工具调用规划

## 📚 今日主题

> **Function Calling 与工具调用规划**（AI / LLM 工程实战）

### 1. 核心概念速览
Function Calling 是 LLM 在生成任务中不直接产出自然语言答案，而是按预定义的结构化 Schema 输出一次或多次“待执行工具调用意图”的能力。其本质是把不可执行的自然语言生成，约束为可被宿主程序解析、验证并执行的中间表示，通常以 JSON 形式表达函数名、参数对象和调用上下文。工具调用规划则是在多步骤任务中，由模型或运行时决定调用顺序、参数来源、依赖关系、失败重试与终止条件的控制流问题。
它解决的核心问题是：LLM 只具备文本生成和概率建模能力，不能直接访问数据库、API、文件系统、搜索引擎或执行代码；Function Calling 将外部能力以声明式接口暴露给模型，使模型成为“调用策略生成器”，宿主进程成为“执行器与状态管理器”。
在 AI 工程体系中，它位于 Prompt/Context 层与执行层之间：上层是任务目标和上下文，下层是真实副作用。它不是模型真的执行函数，而是模型根据上下文输出符合 ABI 约定的调用描述，由程序执行后再把结果写回上下文。
对专业工程师必须掌握的原因在于：Agent、RAG 增强、自动化运维、代码助手、多模态工具链、MCP/A2A 类协议，最终都会收敛为 schema 约束、上下文管理、状态机执行、幂等性、超时重试、结果注入和安全边界。不理解这一层，容易把 Agent 当成黑盒智能体，无法调试、无法做可观测性、无法控制成本和风险。

### 2. 底层原理剖析
底层机制可以拆成五个阶段：
1. 工具注册：宿主把工具描述注入请求，通常包含 name、description、parameters JSON Schema、可选返回结构。description 和 schema 会进入模型上下文，影响模型是否选择该工具以及如何填参。
2. 模型推理：模型基于 system prompt、历史消息、用户输入、工具列表，生成下一段 token。若被训练或约束为工具调用模式，它不会自由输出文本，而是生成结构化工具调用对象。
3. 宿主解析与校验：运行时解析 tool_calls，校验函数名是否存在、参数是否符合 schema、类型是否合法、枚举是否匹配、必填字段是否完整。模型输出可能不合法，因此必须在宿主侧做 validation。
4. 工具执行：宿主在真实进程内调用函数、HTTP、SDK、数据库或子进程。模型没有执行权限，也没有内存、网络、文件访问能力；执行结果来自宿主环境。
5. 结果回注与继续规划：执行结果作为 tool/observation 消息追加到上下文，模型基于新证据决定继续调用、修正参数、并行调用，或直接生成最终回答。

伪代码流程：
request = { messages, tools }
loop:
  response = llm.generate(request)
  if response.has_tool_calls:
    for call in response.tool_calls:
      schema = registry.get(call.name)
      args = validate_json(call.arguments, schema)
      result = executor.invoke(call.name, args)
      messages.append(tool_result(call.id, result))
    continue
  else:
    return response.text

和前端/工程已有概念对比：
1. 与 TS interface 相比：TS 接口是编译期静态契约，类型错误在 tsc 或编辑器阶段被阻断；Function Calling 的 schema 是运行时弱约束，模型是概率生成器，可能生成不符合 schema 的 JSON，必须运行时校验。
2. 与 RPC stub 相比：传统 RPC 是程序员显式调用，参数由代码构造；Function Calling 是模型生成调用意图，参数由上下文推理得到，因此存在幻觉、参数错填、函数选错、重复调用等不确定性。
3. 与事件驱动相比：工具调用类似事件触发副作用，但触发源不是用户输入或定时器，而是模型输出；它需要额外的 guardrail、审计、超时、幂等和取消机制。
4. 与函数式纯计算相比：LLM 调用本身近似纯文本变换，但工具执行是有副作用的；规划层必须区分“模型可重试”和“副作用操作不可随意重试”。
关键认知：模型输出的不是函数调用，而是一个可序列化的调用计划；宿主程序才是真正的执行者。规划质量取决于工具描述质量、上下文压缩质量、schema 约束强度和执行结果的可解释性。

### 3. 基础代码与实战验证
```text
// 极简 Node.js/TypeScript 伪实现：不依赖 AI SDK，只演示 Function Calling 运行时协议。
// 真实场景中 requestLLM 可替换为任意模型 API 调用。

type ToolDef = {
  name: string;
  description: string;
  parameters: Record<string, unknown>; // JSON Schema
};

type ToolCall = {
  id: string;
  name: string;
  arguments: Record<string, unknown>; // 模型生成的参数，未校验前不可信。
};

type ChatMessage =
  | { role: 'system' | 'user' | 'assistant'; content: string }
  | { role: 'tool'; tool_call_id: string; content: string };

const tools: ToolDef[] = [
  {
    name: 'get_weather',
    description: '根据城市名查询当前天气。输入必须是城市名字符串。',
    parameters: {
      type: 'object',
      properties: { city: { type: 'string' } },
      required: ['city'],
      additionalProperties: false, // 严格模式：禁止模型注入未知字段。
    },
  },
];

// 工具注册表：运行时根据函数名定位真实执行逻辑。
const executors: Record<string, (args: any) => Promise<string>> = {
  async get_weather({ city }) {
    // 这里是真实副作用位置：可调用 HTTP、数据库、文件系统等。
    return JSON.stringify({ city, tempC: 21, condition: 'cloudy' });
  },
};

// 极简校验：真实项目应使用 Ajv/Zod/JSON Schema validator。
function validate(call: ToolCall): void {
  const tool = tools.find(t => t.name === call.name);
  if (!tool) throw new Error(`模型调用了未注册函数: ${call.name}`);
  const required = (tool.parameters.required as string[]) ?? [];
  for (const key of required) {
    if (!(key in call.arguments)) {
      throw new Error(`${call.name} 缺少必填参数: ${key}`);
    }
  }
}

// 模拟 LLM：实际返回模型生成的 tool_calls 或最终文本。
async function requestLLM(messages: ChatMessage[], tools: ToolDef[]) {
  const last = messages[messages.length - 1];
  if (last.role === 'user') {
    return {
      tool_calls: [{ id: 'call_1', name: 'get_weather', arguments: { city: 'Hangzhou' } }],
    };
  }
  // 当上下文已有 tool result 时，模型生成最终回答。
  return { content: '杭州当前约 21 度，多云。' };
}

async function run(input: string): Promise<string> {
  const messages: ChatMessage[] = [
    { role: 'system', content: '你只能通过工具获取事实，工具结果必须作为后续推理依据。' },
    { role: 'user', content: input },
  ];

  // Agent loop：模型生成 -> 校验 -> 执行 -> 回注结果 -> 再推理。
  for (let i = 0; i < 5; i++) {
    const res = await requestLLM(messages, tools);

    if (!res.tool_calls) return res.content ?? '';

    messages.push({ role: 'assistant', content: JSON.stringify(res.tool_calls) });

    for (const call of res.tool_calls) {
      validate(call); // 模型输出是概率结果，执行前必须做契约校验。
      const result = await executors[call.name](call.arguments); // 宿主进程真正执行副作用。
      messages.push({ role: 'tool', tool_call_id: call.id, content: result }); // 结果回注上下文，供下一轮推理。
    }
  }

  throw new Error('工具调用超过最大轮次，规划未收敛。');
}

// run('杭州天气如何？') 会先产生工具调用，再把工具结果交给模型生成最终答案。
```

### 4. 常见误区与进阶思考
误区一：认为模型真的“执行”了函数。模型只生成结构化调用意图，所有执行、鉴权、事务、副作用都在宿主进程。若把模型输出直接当可信指令执行，会产生注入、越权、误删、资金风险等问题。正确做法是把模型输出视为不可信的外部输入，执行前必须做 schema 校验、权限校验、参数白名单和副作用分级。
误区二：把工具描述当成可选项，随便写 description 或 schema。模型选择工具和填参数主要依赖上下文中的工具描述；description 模糊、参数命名歧义、schema 不严格，会导致选错工具、幻觉参数、重复调用和规划漂移。正确做法是用工程接口思维设计工具：明确输入、输出、错误语义、幂等性、超时、副作用范围和示例约束。
进阶思考题：在一个工具集中，有“查询订单”的只读工具和“取消订单”的写工具。若模型连续生成两次取消同一订单的 tool_call，你的运行时应该如何在协议层区分“模型重复生成”“用户确认不足”“工具幂等成功”“工具执行失败但模型误以为成功”这四种情况？请从 tool_call_id、业务幂等键、错误消息回注格式、人工确认步骤和状态机终止条件五个维度设计一套可调试、可回放的执行协议。
