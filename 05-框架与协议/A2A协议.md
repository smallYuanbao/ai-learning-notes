# A2A协议

A2A（Agent-to-Agent）协议是一个开放标准，旨在让不同来源、不同框架构建的AI智能体能够相互通信、协作和调用。

## 1. 为什么需要 A2A

MCP 是AI应用与外部工具之间的标准化通信。但还有一种场景，多个AI Agent 之间如何通信、协作。

A2A 协议就是为了解决这个问题而设计的---它定义了Agent之间如何发现彼此、如何发送任务、如何协商和返回结果。

**用你已经熟悉的类比：**

| 你熟悉的 | A2A 里的对应 |
|------|------|
| 前端的微前端架构（多个子应用通信） | 多个 Agent 之间通过 A2A 协议通信 |
| MCP：AI 应用 → 外部工具 | A2A：Agent → Agent |
| RESTful API 调用 | A2A 的任务派发和结果返回 |

> 一句话总结：MCP 是”AI 应用调用工具”的协议，A2A 是”Agent 之间协作”的协议。两者互补，都是 Agent 生态的标准基础设施。


## 2. A2A 的核心概念

- Agent Card（智能体名片）：每个遵循A2A协议的智能体都会发布一个`agent-card.json`文件。这相当于智能体的“数字卡片”或“简历”，详细描述了其能力、技能、端点地址、认证方式等信息。其他智能体可以通过发现并读取这张名片，来了解如何与智能体进行交互。

- Task（任务）：智能体之间写作的基本单元。一个客户端智能体可以通过创建一个“任务”，来请求另一个服务端智能体执行特定的工作，每个任务都有唯一的ID，并会经历从创建到完成（或失败）的完整生命周期。

- Skill（技能）：在Agent Card中定义，描述了智能体可以执行的特定功能，包括技能ID、名称、描述、输入输出格式等。这是其他智能体调用的“接口”。

- Streaming（流式传输）：A2A支持服务器发送事件（SSE），允许智能体实时、流式返回结果，这对于需要快速反馈的交互场景至关重要。

- Message（消息）：客户端 Agent 和服务端 Agent 之间的 Task（任务）请求和响应（JSON-RPC 2.0）。

- Artifact（产物）：通常是服务端 Agent 在完成任务后，返回给客户端的 Result 附件（如生成的图片或文档）。

| 概念 | 一句话解释 | 和你的项目类比 |
|------|------|------|
| Agent Card | Agent 的”名片”，描述自己能做什么、在哪能找到自己 | 你的 `WeatherSkill` 描述——告诉别人”我能查天气” |
| Task（任务） | Agent 之间传递的工作单元 | `unified_agent` 收到用户问题后决定”需要调天气工具” |
| Message（消息） | Agent 之间的通信载体 | `multi_agent_pipeline` 中函数之间的传参 |
| Artifact（产物） | 任务执行后产出的具体成果 | 天气工具返回的温度数据、检索 Agent 返回的文档列表 |


### 2.2 A2A 的核心工作流程

1. 发现：Agent A 通过 Agent Card 发现 Agent B 的能力。

2. 派发：Agent A向 Agent B发送 Task（包含指令和输入数据）

3. 执行：Agent B执行任务。

4. 返回：Agent B 返回 Artifact（执行结果）给 Agent A。


## 3. 技术架构

A2A协议建立在成熟的互联网技术之上，确保了其开放性和互操作性：

- 通信模式：遵循经典的客户端-服务器模式，一个智能体可以通过客户端请求服务，也可以作为服务器提供服务。

- 传输协议：基于HTTP协议进行通信。

- 消息格式：使用`JSON-RPC 2.0`作为标准的数据交换格式。

- 认证机制：遵循 Open API规范进行身份验证，通过在HTTP请求头中传递令牌等方式确保通信安全。

## 4. A2A 和 MCP 详细对比


### 4.1 模型上下文协议（MCP） 

模型上下文协议（MCP）定义了AI智能体如何与个体工具和资源（如数据库或API）交互和利用。

该协议提供以下能力：

- 标准化AI模型和智能体连接并与工具、API和其他外部资源交互的方式。
- 定义描述工具能力的结构化方式，类似于大语言模型中的函数调用。
- 向工具传递输入并接收结构化输出。
- 支持常见用例，如LLM调用外部API、智能体查询数据库，或智能体连接到预定义函数。

### 4.2 Agent2Agent协议（A2A） 

Agent2Agent协议专注于使不同智能体能够相互协作以实现共同目标。

该协议提供以下能力：

- 标准化独立的、通常不透明的AI智能体如何作为对等体进行通信和协作。
为智能体提供应用层协议来相互发现、协商交互、管理共享任务，以及交换对话上下文和复杂数据。
- 支持典型用例，包括客服智能体将查询委托给计费智能体，或旅行智能体与航班、酒店和活动智能体协调。

### 4.3 为什么需要不同的协议？

MCP和A2A协议对于构建复杂AI系统都是必要的，它们解决了不同但高度互补的需求。A2A和MCP之间的区别取决于智能体与什么进行交互。

- 工具和资源（MCP领域）：

    - 特征：这些通常是具有定义明确、结构化输入和输出的原语。它们执行特定的、通常是无状态的功能。例如计算器、数据库查询API或天气查询服务。
    - 目的：智能体使用工具来收集信息并执行离散功能。
- 智能体（A2A领域）：

    - 特征：这些是更自主的系统。它们推理、规划、使用多个工具、在较长的交互中维护状态，并进行复杂的、通常是多轮对话来实现新颖或不断发展的任务。
    - 目的：智能体与其他智能体协作来处理更广泛、更复杂的目标。


- MCP（模型上下文协议）：解决的是智能体与外部世界的连接问题，它标准化了AI模型应该如何调用外部工具（如数据库、API、搜索引擎），是智能体的“万能插头”。

- A2A（Agent2Agent协议）：解决的是智能体与智能体之间的协作问题。它让不同的AI智能体能够互相通信和协调，像一个团队一样合作。

| 对比维度 | MCP | A2A |
|------|------|------|
| 通信双方 | AI 应用 ↔ 工具/数据源 | AI Agent ↔ AI Agent |
| 核心问题 | “怎么让 LLM 调用外部工具？” | “怎么让多个 Agent 分工协作？” |
| 协议定位 | 工具调用标准化 | Agent 间通信标准化 |
| 你的项目实践 | 已通过 MCPClient 接入天气 Server | `multi_agent_pipeline` 可看作简易版 A2A |
| 典型场景 | 查询天气、搜索网页、读写文件 | 检索 Agent→生成 Agent→审核 Agent |

| 方面 | A2A 协议 | MCP 协议 |
|------|------|------|
| 主要用途 | 智能体间协作 | 智能体-工具集成 |
| 交互对象 | 自主智能体 | 工具和资源 |
| 复杂性 | 多轮对话、状态管理 | 单次函数调用 |
| 状态性 | 有状态交互 | 通常无状态 |
| 自主性 | 高度自主的参与者 | 被动的工具执行 |
| 协商能力 | 支持协商和澄清 | 结构化输入/输出 |
| 用例示例 | 客服委托给专家 | 数据库查询、API 调用 |


## 5. Agent Card

Agent Card 包含以下关键信息：

- 基本信息：Agent 的名称(name)、描述(description)、服务URL(url)、提供者信息(provider)、版本(version)、文档链接(documentationUrl)

- 能力(Capabilities)：Agent 支持的可选能力，如是否支持流式传输(Streaming)、推送通知(pushNotifications)、状态转化历史(stateTransitionHistory)。

- 认证要求(Authentication)：Agent 所需的认证方案(如 Basic、Bearer)和凭证信息。

- 默认交互模式(Default Modes)：Agent 在所有技能中默认支持的输入(defaultInputModes)和输出(defaultOutputModes)的MIME类型。

- 技能(skills)：Agent 能执行的具体能力单元。每个技能包含：
    - 唯一标识符 (id)
    - 名称 (name)
    - 描述 (description)
    - 标签 (tags)
    - 示例 (examples)
    - 特定于技能的输入/输出模式 (inputModes/outputModes) (如果与默认不同)

```typescript

interface AgentCard {
  // Agent 的人类可读名称 (例如 "食谱 Agent")
  name: string;
  // Agent 功能的人类可读描述
  description: string;
  // Agent 托管的 URL 地址
  url: string;
  // Agent 的服务提供商信息
  provider?: {
    organization: string;
    url: string;
  };
  // Agent 的版本 (格式由提供商定义，例如 "1.0.0")
  version: string;
  // Agent 文档的 URL
  documentationUrl?: string;
  // Agent 支持的可选能力
  capabilities: {
    streaming?: boolean; // 如果 Agent 支持 SSE
    pushNotifications?: boolean; // 如果 Agent 可以向客户端推送更新通知
    stateTransitionHistory?: boolean; // 如果 Agent 暴露任务的状态变更历史
  };
  // Agent 的认证要求 (旨在匹配 OpenAPI 认证结构)
  authentication: {
    schemes: string[]; // 例如 Basic, Bearer
    credentials?: string; // 客户端用于私有 Card 的凭证
  };
  // Agent 在所有技能中支持的默认交互模式
  defaultInputModes: string[]; // 支持的输入 MIME 类型
  defaultOutputModes: string[]; // 支持的输出 MIME 类型
  // Agent 可以执行的能力单元集合
  skills: {
    id: string; // 技能的唯一标识符
    name: string; // 技能的人类可读名称
    description: string; // 技能描述
    tags: string[]; // 描述技能能力类别的标签 (例如 "烹饪", "客户支持")
    examples?: string[]; // 技能可以执行的示例场景或提示 (例如 "我需要一个面包的食谱")
    inputModes?: string[]; // 技能支持的输入 MIME 类型 (如果与默认不同)
    outputModes?: string[]; // 技能支持的输出 MIME 类型 (如果与默认不同)
  }[];
}
```

## 6. Task

### 6.1 任务的生命周期与交互 

- 创建: 任务总是由客户端创建。
- 状态管理: 任务的状态由远程代理（Server）决定和维护。
- 会话关联: 多个任务可以通过可选的 sessionId 归属于同一个会话，方便管理相关的交互。
- 代理行为: 收到任务请求后，代理可以采取多种行动：
    - 立即满足请求
    - 安排稍后执行的工作
    - 拒绝请求
    - 协商不同的执行方式
    - 向客户端索要更多信息
    - 委派给其他代理或系统
- 持续交互: 即使任务目标初步达成，客户端仍可在同一任务上下文中请求更多信息或进行修改（例如：“画一只兔子”，然后"把它画成红色"）。
- 信息传递: 任务不仅用于传递最终结果（工件）和交互消息（思考、指令等），还维护着任务的状态及其可选的历史记录（状态变化和消息记录）。这对于支持多轮对话式的 AI 交互至关重要。

### 6.2 任务状态

任务具有明确定义的状态，表示其在生命周期中所处的阶段：

- submitted: 任务已提交。
- working: 任务正在处理中。
- input-required: 任务需要客户端提供额外输入。
- completed: 任务已成功完成。
- canceled: 任务已被取消。
- failed: 任务执行失败。
- unknown: 任务状态未知。


完整可参考代码

```typescript
// 任务主体
interface Task {
  id: string; // 任务的唯一标识符
  sessionId: string; // 客户端生成的会话 ID
  status: TaskStatus; // 任务当前状态
  history?: Message[]; // 消息历史记录
  artifacts?: Artifact[]; // 代理创建的工件集合
  metadata?: Record<string, any>; // 扩展元数据
}

// 任务状态及附带消息
interface TaskStatus {
  state: TaskState;
  message?: Message; // 向客户端提供的额外状态更新
  timestamp?: string; // ISO 日期时间值
}

// 任务状态枚举
type TaskState =
  | "submitted"
  | "working"
  | "input-required"
  | "completed"
  | "canceled"
  | "failed"
  | "unknown";

// 用于创建、继续或重启任务的客户端参数
interface TaskSendParams {
  id: string; // 任务 ID
  sessionId?: string; // 会话 ID (如果未设置，服务器会为新任务创建)
  message: Message; // 发送的消息
  historyLength?: number; // 要检索的最近消息数量
  pushNotification?: PushNotificationConfig; // 断开连接时服务器发送通知的配置
  metadata?: Record<string, any>; // 扩展元数据
}

// 服务器在 sendSubscribe 或 subscribe 请求期间发送的状态更新事件
interface TaskStatusUpdateEvent {
  id: string;
  status: TaskStatus;
  final: boolean; // 指示事件流是否结束
  metadata?: Record<string, any>;
}

// 服务器在 sendSubscribe 或 subscribe 请求期间发送的工件更新事件
interface TaskArtifactUpdateEvent {
  id: string;
  artifact: Artifact;
  metadata?: Record<string, any>;
}

// 推送通知配置 (具体结构未在源文档中详细说明)
interface PushNotificationConfig {
  // ... 配置细节
}
```

## 7. Artifact 

### 7.1 Artifact 特性

- 不可变性: 一旦生成，工件的内容通常是不可变的。
- 命名: 工件可以被命名 (name)，方便识别。
- 多部分: 一个工件可以包含多个部分 (parts)，每个部分有自己的内容和类型。
- 流式支持: 通过流式响应 (append: true)，可以将新的部分 (Part) 附加到现有的工件中，适用于逐步生成内容或大数据传输的场景。
- 多工件任务: 一个任务可以生成多个工件。例如，一个"创建网页"的任务可能会生成一个 HTML 工件和一个或多个图像工件。

### 7.2 工件接口定义 

```typescript
interface Artifact {
  name?: string; // 工件的可选名称
  description?: string; // 工件的可选描述
  parts: Part[]; // 工件包含的部分数组
  metadata?: Record<string, any>; // 扩展元数据
  index: number; // 工件在其所属任务中的索引
  append?: boolean; // 是否允许追加 Part (用于流式传输)
  lastChunk?: boolean; // 是否是流式传输的最后一块
}
```

## 8. A2A 的设计原则

- **拥抱智能体能力 (Embrace Agentic Capabilities)**：这是A2A最核心的原则。它允许智能体在不共享内存、工具或上下文的情况下，以各自自然、非结构化的方式进行协作。这意味着每个智能体都是一个独立的“黑盒”，只需通过标准化的接口交互，保护了各自的内部逻辑和数据。

- **基于现有标准构建 (Build on Existing Standards)**：A2A没有重新发明轮子，而是直接构建在广泛使用的成熟技术之上，例如 HTTP、JSON-RPC 2.0 和 Server-Sent Events (SSE)。这极大地降低了学习和集成的门槛，使其能轻松融入现有的IT系统。

- **默认安全 (Secure by Default)**：协议在设计之初就考虑了企业级的安全需求。它原生支持企业级的身份验证和授权机制，确保只有经过授权的用户和系统才能访问智能体。

- **支持长时间运行的任务 (Support for Long-Running Tasks)**：A2A原生支持异步优先（Asynchronous First） 的工作模式。它可以灵活处理从秒级快速响应到需要数小时甚至数天（例如需要人工介入审批）的复杂任务，并能在过程中提供实时反馈、通知和状态更新。

- **模态无关 (Modality Agnostic)**：A2A不仅限于文本交互，它支持多种数据模态，包括音频、视频流、表单（form）、内嵌框架（iframe） 等。这使得智能体能够处理更丰富、更复杂的交互场景。

- **简洁性 (Simplicity)**：通过优先采用HTTP、JSON-RPC等现有标准，A2A确保了协议本身的简洁性，避免了不必要的复杂性。

- **企业就绪 (Enterprise Readiness)**：A2A在设计时全面考虑了企业级应用的需求，包括认证、授权、安全、隐私、追踪和监控等方面，并遵循标准的Web实践。


