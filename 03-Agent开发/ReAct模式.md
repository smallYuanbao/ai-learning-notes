# ReAct模式

> 📅 学习日期：2026-07-21 ~ 2026-07-22
> 🔗 关联面试题：Agent 专题 题11（Agent 核心工作流程）、题13（动态 API 调用的实现方式）、题17（Agent 死循环问题的解决方法）

## 为什么需要ReAct？

ReAct模式是当前AI Agent领域最基础、也最具代表性的设计范式

这个名字本身就是它的核心思想：ReAct = Reasoning（推理） + Acting（行动

在ReAct出现之前，主要有两种思路：

- 纯推理（CoT，思维链）：让LLM一步一步地“思考”来解决问题，但它无法与外部世界交互，也无法获取实时信息。

- 纯行动（Act-Only）：让LLM直接执行动作，但缺少中间的推理过程，像个“黑盒”，一旦出错很难调试和修正。

**ReAct的核心目标，就是将“推理”和“行动”两者结合起来，让AI从一个“被动应答者”升级为“主动问题解决者”。**

## ReAct 的核心循环

**ReAct的核心循环：Thought -》 Action -》Observation。**


1. **思考（Thought）**
- **动作主体**：LLM
- **主要任务**：分析当前状况，决定下一步策略。
- **具体表现**：模型生成一段文本，解释自己为什么要这样做。
- 这一步不改变外部环境，是模型的“内心独白”。 

2. **行动（Action）**
- **动作主体**：LLM（大模型）输出参数 -》 用户系统执行
- **主要任务**：选择要使用的工具，并给出具体参数。
- **具体表现**：输出特定格式的指令或Funtion Calling参数。
- 这一步会改变环境或获取信息。

3. **观察（Observation）**
- **动作主体**：外部环境 / 你的后端系统
- **主要任务**：将工具执行的真实结果返回给模型。
- **具体表现**：系统把API、数据库或代码运行的真实输出拼接进上下文。
- 这个结果会成为下一轮“思考”的依据。


## ReAct 和 Function Calling的区别


**最核心的区别：Function Calling 是大模型的一项“原子能力“（让模型输出JSON参数），而ReAct是一种“工作流算法”（教模型如何利用这种能力去思考和多步循环）**

**ReAct是”战略指挥“（工作流程范式），Function Calling 是”战术动作“（具体实现手段）**

| 维度 | Function Calling | ReAct |
|------|:---:|:---:|
| 本质定位 | 模型底层能力 / API 特性（原子能力） | Prompt 架构 / Agent 设计模式（工作流算法） |
| 思维过程 | **无 Thought**，直接输出 JSON | **必须有 Thought**，先思考再行动 |
| 运行机制 | **单次交互**：输入 → JSON → 结束 | **多轮循环**：思考 → 行动 → 观察 → 再思考 |
| 核心产出 | tool_calls JSON（函数名+参数） | Thought → Action → Observation 循环轨迹 |
| 实现层级 | 厂商底层微调 | 应用层代码 / 框架控制 |
| 执行主体 | LLM 生成 JSON + 后端执行函数 | 编排器（Orchestrator）控制循环流程 |
| 关系 | 是 ReAct 中最常见的 Action 实现手段 | 可以用 Function Calling 实现 Action，也可以不用 |


## ReAct prompt格式

### 流派一：经典纯文本版（适用于不支持 Function Calling 的开源模型，如 Llama 3、Qwen）

这是 ReAct 原始论文中的标准写法，核心思想是用文本格式约束模型的输出。

1. Synstem Prompt（系统指令）

你需要把这段指令放在 system 消息中，告诉模型它的身份、拥有的工具以及必须遵循的输出格式。

```text
你是一个智能助理，你可以使用以下工具来帮助用户解决问题。你必须严格遵循以下格式回答：

工具列表：
- search(query: str): 用于搜索实时信息。
- get_weather(city: str): 用于查询指定城市的天气。
- calculate(expression: str): 用于计算数学表达式。

你必须使用以下格式进行回应：

Thought: 你当前对问题的思考和推理过程。
Action: 你需要调用的工具名称，以及传递给工具的 JSON 格式参数。
Observation: （这是系统预留的，用于接收工具返回的结果，你自己不要生成这一行）

……（当你有足够信息时，停止以上循环，使用以下格式输出）……
Thought: 我已经掌握了足够的信息，可以回答用户了。
Final Answer: 你最终对用户的完整回复。
```

2. User Prompt 与 过程管理（关键 ！）

当用户提问后，我们通常会在 User 消息末尾强制加上一句前缀，引导模型从“思考”开始，而不是直接输出答案：

```text
用户提问：{用户的原始问题}

请开始你的推理和行动。首先输出你的 Thought：
```

3. 运行时动态拼接（解析逻辑）

当模型输出了 Action: get_weather({"city": "北京"}) 后，后端代码解析并执行工具，拿到 "25°C"。此时，后端不会把 Observation 作为新的一轮 User 消息，而是强制插入到当前消息的下方，拼接成：

```text
Observation: 25°C
```
然后带着“上一轮 Assistant 的 Action” + “拼接的 Observation” 再次请求模型，模型就会接着输出下一轮的 Thought: ...，直到它输出 Final Answer 为止。


完整的prompt内容：

```text
你是一个智能助理，你可以使用以下工具来帮助用户解决问题。你必须严格遵循以下格式回答：

工具列表：
- get_weather(city: str): 用于查询指定城市的实时天气信息。

你必须使用以下格式进行回应：

Question: 用户输入的问题
Thought: 你当前对问题的思考和推理过程。
Action: 你需要调用的工具名称，以及传递给工具的 JSON 格式参数。
Observation: （这是系统预留的，用于接收工具返回的结果）

……（当你有足够信息时，停止以上循环，使用以下格式输出）……
Thought: 我已经掌握了足够的信息，可以回答用户了。
Final Answer: 你最终对用户的完整回复。

开始！ 

Question: 北京今天天气怎么样？
Thought: 用户想知道北京今天的天气，我需要调用天气查询工具。
Action: get_weather
Action Input: {"city": "北京"}
Observation: 北京今天晴，气温 25°C，微风。
```

其中：开始的作用是充当对话模式与执行模式之间的硬性边界分隔符和强制启动触发器。

### 流派二：原生 Function Calling 版（适用于 DeepSeek/GPT 系列）

1. System Prompt（核心指令）

工具的元数据（名称、参数）不再写在 System Prompt 里，而是通过 API 的 tools 参数传递。System Prompt 只需要控制模型的思考节奏和终止条件：

```text
你是一个功能强大的 AI 助手。你可以根据用户问题调用外部工具来获取实时信息。

当你接收到用户的请求时，请遵循以下步骤：
1. 首先，在内部进行推理（无需输出 Thought 文本），判断是否需要调用工具。
2. 如果需要调用工具，请使用标准的 tool_calls 功能输出调用指令。
3. 当你收到工具返回的 Observation（观察结果）后，必须进行反思：
   - 如果信息足以回答用户，则直接生成最终回复。
   - 如果信息不足，请再次发起 tool_calls 补充信息。

注意：
- 如果你是调用工具，请输出标准的函数调用 JSON。
- 如果你是在回答用户，请直接输出自然语言，不要包含 "Final Answer:" 等无意义的前缀。
- 如果你发现工具返回了错误信息，请尝试换个参数重试，或告知用户无法完成。
```
2. 运行时的消息结构

在这个模式下，你的 messages 数组会随着循环动态增长，形成一个完整的“思考-行动”轨迹（无需手动解析文本）：

```text
[
  { "role": "system", "content": "你是一个功能强大的 AI 助手..." },
  { "role": "user", "content": "北京和上海哪个更热？" },
  { "role": "assistant", "content": null, "tool_calls": [{"function": {"name": "get_weather", "arguments": "{\"city\":\"北京\"}"}}] },
  { "role": "tool", "tool_call_id": "call_123", "content": "26°C" },
  { "role": "assistant", "content": null, "tool_calls": [{"function": {"name": "get_weather", "arguments": "{\"city\":\"上海\"}"}}] },
  { "role": "tool", "tool_call_id": "call_456", "content": "29°C" }
]
```

### 模版中的”黄金比例“

无论你用哪个流派，Prompt 模板都需要包含三个必不可少的隐性指令（很多新手会忽略）：

1. 必须强调“只调必需的工具”：加一句“如果你能直接回答，绝不要调用工具”，否则模型会为了“炫技”什么事都去查 API，增加延迟和成本。

2. 必须暗示“可调用多次”：明确告诉模型“你可以分步获取信息”，避免模型试图在一个 tool_calls 里完成所有事，导致参数过于复杂而出错。

3. 必须设定“异常退路”：加入“如果你尝试 3 次工具调用后依然没有结果，请告诉用户查询失败”。


## ReAct 各个流程的格式

### 纯文本

1. 核心字段的“强制性语法”

| 字段 | 前缀 | 格式规则 | 示例 |
|------|------|---------|------|
| 思考 | `Thought:` | 英文半角冒号 + 空格，内容任意自然语言（可多行） | `Thought: 用户想查北京天气，我需要调用工具。` |
| 行动（工具名） | `Action:` | 英文半角冒号 + 空格，必须精准匹配定义的工具名（大小写敏感） | `Action: get_weather` |
| 行动（参数） | `Action Input:` | ⚠️ 中间有空格，冒号后跟空格，参数通常为 JSON 字符串 | `Action Input: {"city": "Beijing"}` |
| 终止回答 | `Final Answer:` | 当模型决定不再调用工具时输出，后面跟最终回复 | `Final Answer: 北京今天25°C。` |

2. 一个“绝对标准”的完整轨迹（含换行符）

模型输出的原始文本（原始字符串）必须是这样：

```text
Thought: 用户想知道北京今天的天气，但我没有实时数据，需要调用工具。
Action: get_weather
Action Input: {"city": "北京"}
```
特别注意 Action Input 的换行细节：

- Action: 后面不要直接跟参数，必须换行写 Action Input:。

- Action Input: 后面的 JSON 必须写成一行，不能为了美观而换行（除非你的解析器支持多行 JSON 合并）。

3. 关于 Observation 的特殊约定（极易踩坑）

在格式定义中，Observation: 不是让 LLM（大模型）去生成的，而是后端代码在拿到工具执行结果后，强行“拼接”进上下文的。

- LLM 输出的结尾：一定是 Action Input: {...}（等待工具结果）。

- 后端拼接后发给 LLM 的内容：在上一轮的 Action Input 后面，强行加上一行 Observation: 工具返回的真实数据。

格式如下：

```text
Thought: 用户想知道北京今天的天气，我需要调用工具。
Action: get_weather
Action Input: {"city": "北京"}
Observation: 北京今天晴，气温 25°C，微风。
```
大模型拿到Observation后，继续输出下一轮：

```text
Thought: 我已经拿到了天气数据，可以回答了。
Final Answer: 北京今天晴天，25°C。
```
4. 参数格式的“两种流派”（解析器要兼容）

建议在 System Prompt 里强制要求使用 JSON 格式，并在后端解析时用 json.loads() 处理，这样扩展性最强。

```python
import json

def parse_action_input(raw_input: str):
    raw_input = raw_input.strip()
    try:
        # 尝试标准 JSON 解析
        return json.loads(raw_input)
    except json.JSONDecodeError:
        # 💥 触发解码异常，说明大模型直接吐了裸文本（如 "北京"）
        # 此时由代码人工为其组装成标准结构，保障下游业务不崩溃
        return {"city": raw_input, "date": "tomorrow"}  # 结合业务补全默认参数
```

5. 解析器的“硬边界匹配规则”（后端代码怎么抠？）

你的 Python 正则表达式必须按以下优先级来截取：

- 匹配 Final Answer：如果存在，直接截取后面的所有内容，终止循环（最高优先级）。

- 匹配 Action：提取工具名。

- 匹配 Action Input：提取参数（注意：参数内容可能跨行，必须用 re.DOTALL 匹配，直到遇到下一个 \n[A-Z] 或文本结尾）。

**核心避坑指南（必看）**
1. Action 和 Action Input 必须分成两行：如果写成 Action: get_weather {"city": "北京"}，你的正则会非常难写，且模型容易生成错乱的参数。

2. 区分 Observation 的“使用者”：你发给 LLM 的 Prompt 里可以带上 Observation 的例子，但绝对不能在 System Prompt 里要求 LLM 自己生成 Observation。你必须在代码里拦截并手动注入。

3. 大小写敏感：Thought、Action、Observation、Final Answer 的首字母必须大写，这是原始论文的标准，也是大多数开源模型（如 Llama、Qwen）微调时遵循的格式。


### Function Calling

1. 核心字段映射表（纯文本 vs FC）
| 纯文本 ReAct 字段 | Function Calling 对应物 | 说明 |
|------|------|------|
| Thought（思考） | 不再强制输出文本，由 System Prompt 隐式控制 | 思考过程不参与格式解析，只作自然语言上下文 |
| Action（工具名） | `message.tool_calls[0].function.name` | 标准化 JSON 字段，无需正则匹配 |
| Action Input（参数） | `message.tool_calls[0].function.arguments` | 必须是合法 JSON 字符串，API 强制校验 |
| Observation（观察） | `role: "tool"` 的消息，携带 `tool_call_id` + `content` | 不再是纯文本拼接，而是独立的标准化消息角色 |

2. 具体例子比对

纯文本模式：

```text
Thought: 用户想查北京天气，我要调用工具。
Action: get_weather
Action Input: {"city": "北京"}
```

Function Calling:

模型返回的结构（message部分）：

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "get_weather",
        "arguments": "{\"city\": \"北京\"}"
      }
    }
  ]
}
```
后端直接读取 message.tool_calls[0].function.name 和 .arguments，不需要任何正则表达式，100% 精准。

当你执行完工具后，注入 Observation 的方式也从“拼接纯文本”变成了追加标准消息对象：

```json
{
  "role": "tool",
  "tool_call_id": "call_abc123",
  "content": "北京今天晴，25°C"
}
```

3. ”思考“去哪儿了

在 FC 模式下，思考过程被移到了 System Prompt（系统提示词） 中作为隐性推理指令，而不再作为显式的文本 Token 输出。

你可以这样写 System Prompt（对应之前的 ReAct 逻辑）：

“当你决定调用工具时，直接输出 tool_calls。当你收到工具返回结果后，判断信息是否充足；若充足，直接生成最终回复；若不足，再次发起 tool_calls。”

模型在内部推理时依然会“思考”，但不再把思考过程写成文字浪费 Token，直接输出工具调用或最终答案，这让 FC 模式比纯文本 ReAct 更快、更便宜（省掉了 Thought: ... 的 Token 开销）。

4. 总结

| 对比维度 | 纯文本 ReAct | 原生 Function Calling |
|------|------|------|
| 格式依赖 | 依赖 LLM 严格输出 `Action:` 等文本前缀 | 依赖 API 原生 JSON 结构 |
| 解析方式 | 正则表达式（脆弱、易错） | 直接取 `tool_calls` 字段（强类型、稳定） |
| Token 成本 | 较高（必须生成 Thought 消耗 Token） | 较低（只输出工具调用或最终回答） |
| 适用场景 | 不支持 FC 的开源小模型（如 Llama 2/3） | DeepSeek/GPT/Claude 等商业或微调模型 |
| 工程复杂度 | 需要自己写解析器和拼接逻辑 | 只需处理 API 的标准响应格式 |


## ReAct 的循环终止条件

终止条件分三类，前两种由 LLM 自主判断，第三种由代码硬保障：

### 一、正常终止

**纯文本模式 — 关键词截停**：模型判定信息充足后输出 `Final Answer:`，后端正则匹配到此前缀后截取最终回复并退出循环。

System Prompt 约束：
```text
当你已经获取到足够回答用户问题的所有信息后，你必须停止调用工具。
此时，你的输出必须以 “Final Answer:” 作为开头，后面跟上对用户的最终回复。
绝对禁止在 “Final Answer:” 之后继续输出任何 “Action:” 或 “Thought:”。
```

代码硬匹配：
```python
if “Final Answer:” in model_output:
    final_response = model_output.split(“Final Answer:”)[-1].strip()
    return final_response  # 立即退出循环
```

**Function Calling 模式 — 双重判断**：每次请求后按优先级检查：

| 检查项 | 判定逻辑 | 结论 |
|:---:|------|------|
| ① `finish_reason` | 如果是 `”stop”` 且没有 `tool_calls` | **立即停止**：模型认为任务已完成 |
| ② `message.tool_calls` | 如果是 `None` 或空数组 | **立即停止**：模型在直接回答 |
| ③ `tool_choice: “required”` | 如果代码设置了该参数 | **跳过停止逻辑**：强制调用工具 |

Prompt 约束：
```text
如果你认为当前已掌握的信息足以准确回答用户的问题，请直接输出自然语言回复，不要调用任何函数。
如果你对某个细节不确定，必须先调用工具查询，严禁凭空编造。
```

### 二、异常终止

连续两次 Observation 返回相同错误（如 API 超时），Agent 主动放弃并告知用户”暂时无法完成”。

### 三、硬保险（代码层，不依赖 LLM 自觉）

| 硬保险机制 | 代码实现建议 | 触发后的行为 |
|------|------|------|
| 最大迭代次数 | `MAX_STEPS = 5`（大多数任务 5 轮足够） | 达到后强制 `break`，返回”任务步骤过多，已暂停” |
| 工具重复调用检测 | 记录最近 3 次 Action 的 name+参数，完全一致视为死循环 | 强制中断，提示”检测到重复操作，已停止” |
| 总 Token 限制 | API 请求中设置 `max_tokens` | `finish_reason` 变为 `”length”` 说明被截断，提示用户重试 |
