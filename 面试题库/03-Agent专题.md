# Agent 专题

## ⏳ 待整理清单

### 🟢 已完成（7 道）

| 题号 | 题目 | 知识来源 |
|:---:|------|------|
| 8 | 大模型 Agent 的定义及与传统 AI 系统的区别 | FunctionCalling 笔记 §一 |
| 9 | LLM Agent 的基本架构组成 | FunctionCalling 笔记 §一 |
| 11 | Agent 智能体的核心工作流程 | FunctionCalling 笔记 §一 + 流程图 |
| 13 | LLM Agent 动态 API 调用的实现方式 | FunctionCalling 笔记 §二 |
| 17 | Agent 死循环问题的解决方法 | ReAct 笔记 §终止条件 |
| 18 | AI Agent 与直接调用大模型 API 问答的本质区别 | FunctionCalling 笔记 §四 |
| 19 | Tool Calling 工具调用的完整链路解析 | FunctionCalling 笔记 §二 |

### 🔴 待撰写（41 道）

| # | 分类 | 题号 | 题目 | 解决节点 |
|---|------|:---:|------|---------|
| 1 | 基础 | 10 | LLM Agent 的常见功能介绍 | Agent 学完后 |
| 2 | 基础 | 7 | Copilot 模式与 Agent 模式的核心差异 | Agent 学完后 |
| 3 | 记忆 | 12 | LLM Agent 长期记忆能力的实现方法 | Memory 学完后 |
| 4 | 记忆 | 22 | 短期记忆与长期记忆的差异及存储检索方式 | Memory 学完后 |
| 5 | 框架 | 1 | OpenClaw 的核心原理解析 | 阶段 3 |
| 6 | 框架 | 2-6 | LangChain/LlamaIndex 相关（5 道） | 阶段 4 |
| 7 | 框架 | 14-16 | 多模态推理/框架对比/AutoGPT（3 道） | 阶段 3-4 |
| 8 | OpenClaw | 23-26,28-31,35-45 | OpenClaw 专属（17 道） | 阶段 3 |
| 9 | 通用 | 20-22,27,32-34 | 通用 Agent / Skills / MCP（7 道） | 阶段 3 |
| 10 | LangChain | 46-48 | LangChain Agent（3 道） | 阶段 4 |

> 📊 共 48 道：🟢 7 道 + 🔴 41 道（分 10 组）
> 🔑 下一步：题10（Agent 常见功能）→ Memory（题12/22）→ 框架 & OpenClaw（阶段 3-4）

---

## 题8：大模型 Agent 的定义及与传统 AI 系统的区别

**我的回答**

**定义**：Agent 是一个能自主思考、会用工具、能拆任务的大模型应用。概念层看有五要素——感知、规划、记忆、行动、反思；工程层看就是 Agent = LLM + Memory + Tools + Planning + Action。

**与传统软件和纯 LLM 的核心区别**：

| 维度 | 传统软件 | 纯 LLM | Agent |
|------|---------|--------|-------|
| 驱动方式 | 硬编码规则（If-Else） | 概率预测与文本生成 | 目标驱动 + 自主规划 |
| 灵活性 | 极低，超出规则就报错 | 中等，能理解生成但无行动力 | 极高，能自主摸索解决路径 |
| 工具使用 | 只能调用自带功能 | 无法直接使用外部工具 | 自主决定何时调哪个工具 |
| 角色比喻 | 按说明书组装的机器 | 知识渊博但没手脚的军师 | 能独立完成项目的专业员工 |

**项目关联**：baby-ai 当前是 LLM + RAG 模式，正在向 Agent 架构演进——加入 Function Calling 让模型自主调天气 API、查数据库，从"被动回答"升级为"主动执行任务"。

**关键词**：Agent、大模型智能体、传统软件、纯 LLM、目标驱动、工具调用



## 题9：LLM Agent 的基本架构组成

**我的回答**

Agent = LLM + Memory + Tools + Planning + Action，我分两组理解：

- **基础能力组**（LLM + Memory + Tools）：静态的”弹药库”——LLM 大脑负责推理、Memory 提供对话历史上下文、Tools 是可调用的外部函数列表
- **循环执行组**（Planning → Action → Reflection → 再规划）：动态的”作战流程”——拆任务、调工具、看结果、反思再迭代，直到任务完成

两者配合：基础能力组提供弹药（记忆上下文 + 可用工具），循环执行组据此打仗，Action 的反馈实时回流更新 Memory，形成闭环。

**项目关联**：baby-ai 当前已具备 Memory（对话历史管理）和 Tools（天气 API），正在加入 Planning 和 Reflection 能力，向完整 Agent 架构演进。

**关键词**：Agent 架构、LLM、Memory、Tools、Planning、Action、循环执行、弹药与打仗


## 题11： Agent 智能体的核心工作流程

**我的回答**

Agent 的核心工作流程本质上就是 ReAct（Reasoning + Acting）循环：**感知 → 规划 → 行动 → 观察 → 反思 → 再循环**。

**四个关键环节**：

| 步骤 | 角色 | 做什么 | 实例（"北京明天能带宝宝出门吗"） |
|:---:|------|------|------|
| ① 意图识别 | LLM | 判断用户需求是否需要调工具，决定调用哪个函数 | 判断"需要天气数据"→ 选 get_weather |
| ② 工具调用 | 编排器 | 解析 LLM 返回的 tool_calls，执行对应函数，拿到 Observation | 调用天气 API，拿到"25°C、晴" |
| ③ 结果回传 | 编排器 | 将 Observation 以 tool_message 追加到对话历史，还给 LLM | 把"25°C、晴"注入 Memory |
| ④ 反思生成 | LLM | 基于工具返回的结果 + 原始问题，生成最终回答或决定继续调工具 | "25°C 晴天适合带宝宝出门，建议防晒" |

**这个循环的核心价值**：LLM 不只是一次性输出——它可以"边做边想"，一次工具结果不够就再调一次，直到信息充足才给出最终回答。这和传统软件"一条路走到黑"以及纯 LLM"一次生成"有本质区别。

**项目关联**：baby-ai 正在从纯 RAG 模式向 Agent 模式演进，已实现天气查询的 Function Calling，后续将加入多工具协同和 ReAct 循环。

**关键词**：Agent 工作流程、ReAct、意图识别、工具调用、Observation、反思循环


## 题13：LLM Agent 动态 API 调用的实现方式

**我的回答**

"动态"体现在 LLM 不是代码写死调哪个 API，而是在运行时根据用户语义自主决定：**要不要调、调哪个工具、传什么参数**。

**实现方式分两步**：

**1. 工具注册（开发者做）**

将可用的外部函数按 JSON Schema 格式定义——包括函数名、描述、参数类型和约束——随 LLM 请求一起传入 `tools` 参数。Schema 就是"菜单"，告诉 LLM 有哪些菜可以点。

**2. 运行时动态决策（LLM 做）**

用户提问后，LLM 基于语义理解自主决策：
- 判断是否需要调工具——能直接回答的就不调，避免"为用而用"
- 如果需要，从已注册的工具列表里选最匹配的那个
- 从用户原话中提取或推理参数值（如"北京明天" → city="北京", date="2026-07-23"）
- 输出标准化的 tool_calls JSON，交给后端执行

**和代码写死调 API 的本质区别**：

| | 硬编码调用 | 动态 API 调用 |
|------|------|------|
| 调不调 | 代码 if-else 写死 | LLM 运行时理解意图后决定 |
| 调哪个 | 固定一个函数 | 从多个工具中选出最匹配的 |
| 参数怎么来 | 前端表单/代码硬传 | LLM 从自然语言中提取推理 |

**项目关联**：baby-ai 已实现天气查询的动态 API 调用——get_weather 按 DeepSeek 规范定义 Schema，LLM 在用户问天气相关问题时自动触发调用，后端解析 tool_calls 执行真实 API，Observation 回填后 LLM 综合数据给出育儿建议。

**关键词**：动态 API 调用、Function Calling、JSON Schema、tool_calls、语义理解、工具注册



## 题17：Agent 死循环问题的解决方法

**我的回答**

解决方法分为三类：

1. 正常终止

**纯文本模式** — 关键词截停：模型判断信息充足后输出 `Final Answer:`，后端正则匹配到此前缀后截取最终回复并退出循环。

**Function Calling 模式** — 双重判断，每次请求后按优先级检查：

| 检查项 | 判定逻辑 | 结论 |
|:---:|------|------|
| ① `finish_reason` | 如果是 `”stop”` 且没有 `tool_calls` | **立即停止**：模型认为任务已完成 |
| ② `message.tool_calls` | 如果是 `None` 或空数组 | **立即停止**：模型在直接回答 |
| ③ `tool_choice: “required”` | 如果代码设置了该参数 | **跳过停止逻辑**：强制调用工具 |

2. 异常终止

连续两次 Observation 返回相同错误（如 API 超时），Agent 主动放弃并告知用户”暂时无法完成”。

3. 硬保险（代码层，不依赖 LLM 自觉）

| 硬保险机制 | 代码实现建议 | 触发后的行为 |
|------|------|------|
| 最大迭代次数 | `MAX_STEPS = 5`（大多数任务 5 轮足够） | 达到后强制 `break`，返回”任务步骤过多，已暂停” |
| 工具重复调用检测 | 记录最近 3 次 Action 的 name+参数，完全一致视为死循环 | 强制中断，提示”检测到重复操作，已停止” |
| 总 Token 限制 | API 请求中设置 `max_tokens` | `finish_reason` 变为 `”length”` 说明被截断，提示用户重试 |

**项目关联**：baby-ai 正在向 Agent 架构演进，计划在 ReAct 循环中实现上述三层终止机制——纯文本模式用正则截取 `Final Answer`，FC 模式通过 `finish_reason` + `tool_calls` 双重判断，代码层硬保险设置 `MAX_STEPS = 5` 防止 Token 烧穿。

**关键词**：死循环、Final Answer、finish_reason、tool_calls、最大迭代次数、重复调用检测、Token 限制





## 题18：AI Agent 的定义及与直接调用大模型 API 问答的本质区别

**我的回答**

**定义（同题8）**

**区别：**

直接调用API是“问一句答一句”，Agent是“理解意图-》决定要不要调用工具-》调工具拿结果-》综合生成回答”

**实测对比：**
同样问"北京今天天气怎么样，适合带宝宝出门吗"——普通 /api/chat 只能泛泛而谈，Agent 版调天气 API 拿到真实温度湿度，结合 RAG 知识库给具体建议

**本质差异**：LLM API 是"知识渊博但没有手脚的军师"，Agent 是"能自己动手干活的员工"

**项目关联**：baby-ai 已有实测对比——普通 `/api/chat` 被问到"北京今天天气适合带宝宝出门吗"时只能泛泛而谈，Agent 版本调天气 API 拿到真实温度湿度后结合 RAG 知识库给出具体建议。两张截图证明 Agent 在需要实时数据时明显优于纯 LLM。

**关键词**：AI Agent、LLM API、Function Calling、工具调用、自主决策、实测对比

## 题19：Tool Calling 工具调用的完整链路解析

**我的回答**

Tool Calling 的完整链路分四步：

| 步骤 | 角色 | 动作 | 数据流向 |
|:---:|------|------|:---:|
| ① 注册工具 | 开发者 | 将外部函数按 JSON Schema 格式定义，注册到 LLM 请求的 tools 参数中 | Tools → LLM |
| ② 意图识别 | LLM | 理解用户语义，判断是否需要调工具，若需要则输出结构化 tool_calls 而非文本 | LLM → 编排器 |
| ③ 本地执行 | 编排器 | 解析 tool_calls，路由到对应函数执行，获取 Observation | 编排器 → Tools |
| ④ 结果回传 | 编排器 | 将 Observation 以 tool_message 追加到 Memory，还给 LLM 完成推理闭环 | Memory → LLM |

**关键理解**：LLM 本身不执行任何函数——它只负责"翻译"（把意图转为 JSON），真正的执行在外围代码。执行完后结果必须回填，LLM 才知道"这个工具返回了什么"，才能生成最终回答或决定继续调下一个工具。

**项目关联**：baby-ai 已实现天气查询的 Function Calling，get_weather 工具按 DeepSeek 规范定义 Schema，后端解析 tool_calls 调用真实 API，Observation 回填后 LLM 综合天气数据 + RAG 知识库给出育儿建议。

**关键词**：Tool Calling、Function Calling、JSON Schema、tool_calls、Observation、推理闭环

