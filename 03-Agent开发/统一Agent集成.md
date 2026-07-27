# 统一Agent集成

> 📅 整理日期：2026-07-27
> 🔗 关联面试题：Agent 专题 题10（Agent 常见功能介绍）、题11（核心工作流程）、题35、40、41、42（多 Agent 协作与通信，⚠️ OpenClaw 部分待补）、协议与框架 题8（MCP 与 FC 差异）

## 一、模块全景（做了什么）

| 模块 | 代码位置 | 作用 |
|------|------|------|
| 预处理 | `query_rewrite.py` + `intent_router.py` | 指代消解 + 意图分类 |
| 工具调用 | `agent.py` → `react_agent_chat` | ReAct 循环 + Function Calling |
| MCP 接入 | `agent.py` → `MCPClient` | 通过 MCP 协议调用外部工具 |
| 混合检索 | `rag.py` → `hybrid_search` | Dense + Sparse 混合检索 |
| Rerank | `rerank.py` → `rerank` | 三级降级精排 |
| 生成 | `llm.py` → `call_deepseek` | Prompt 拼接 + LLM 生成 |
| 反思审核 | `agent.py` → `reflect_and_correct` | 外部批评式审查修正 |
| 多 Agent | `agent.py` → `multi_agent_pipeline` | 检索→生成→审核流水线 |


## 二、关键设计决策（为什么这么做）

| 决策点 | 选择 | 原因 |
|------|:---:|------|
| 工具决策放在检索之前还是之后？ | 并行 | 工具和 RAG 同时跑，两路结果合并，不互斥 |
| 有工具数据时是否还需要 RAG 检索？ | 始终跑 | 工具数据（实时）插入最前面，RAG 领域知识作为补充 |
| 审核用原始问题还是改写后的问题？ | 原始 user_message | 用户原始表述是最准确的意图锚点，改写出错时还能纠偏 |
| 意图路由和 QR 是否作为独立 Agent？ | 不作为 | 轻量文本处理（<1ms），不值得独立成 Agent 节点 |
| 工具调用失败时是否阻塞主流程？ | 不阻塞 | 返回 (False, "")，Agent 仍可用 RAG 知识库回答 |
| MCP 不可用时如何兜底？ | MCP 优先 + 本地兜底 | _get_available_tools：MCP 工具列表为空时回退 AVAILABLE_TOOLS |
| 工具决策用单步还是 ReAct 多步？ | ReAct 多步（max_steps=5） | 简单问题一轮结束，复杂问题自动多轮推理 |
| Prompt 用旧版还是优化版？ | 优化版（buildPromptTest） | 条件检查 + 结构化输出 + Few-shot，减少幻觉 |
| 同轮多个 tool_calls 执行方式 | ThreadPoolExecutor 并行 | 同轮内工具互不依赖，线程池并行执行后批量收集结果 |


## 三、执行流程（怎么跑的）

unified_agent_chat

用户问题 (user_message, session_id, client_history, mcp_client)
    │
    ▼
┌─────────────────────────────────────────┐
│ 1️⃣  预处理                               │
│  get_query_rewrite()   → 改写后的消息     │
│  route_intent()        → 意图类别+SP     │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│ 2️⃣  工具决策 check_and_call_tool()       │
│                                         │
│  ① 取 MCP 工具列表                       │
│  ② 调 LLM（带 tools）                    │
│     ↓                                   │
│  ┌─────────────────────────┐            │
│  │ ReAct 循环 (max 5 steps)              │
│  │                                         │
│  │ finish_reason = "stop"  → 结束循环     │
│  │ finish_reason = "tool_calls" →         │
│  │   ThreadPoolExecutor 并行执行工具       │
│  │   ┌─ get_weather("北京","今天") ─┐    │
│  │   └─ get_weather("北京","明天") ─┘    │
│  │   追加 tool 结果到 messages             │
│  │   回到循环顶部                          │
│  │                                         │
│  │ finish_reason = other → 安全退出       │
│  └─────────────────────────┘            │
│                                         │
│  返回 (need_tool, tool_data)             │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│ 3️⃣  RAG 检索 rag_and_rerank()            │
│  hybrid_search_rrf(top_k=20)  → 粗排     │
│  rerank(top_k=5)              → 精排     │
│  返回 (references, context_docs)         │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│ 4️⃣  合并上下文                            │
│                                         │
│  if tool_data:                           │
│    _format_tool_data() → 人类可读文本     │
│    context_docs.insert(0, 天气数据)      │
│                                         │
│  提取 Reference.text → doc_texts         │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│ 5️⃣  生成回答                              │
│  _build_messages() → messages 数组       │
│  generation_agent() → initial_answer    │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│ 6️⃣  审查反思 review_agent()               │
│  REFLECTION_PROMPT + 原始问题 + 初始回答  │
│  → LLM 找出问题 + 输出修正后完整回答       │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│ 7️⃣  返回                                  │
│  { answer, references, category }       │
└─────────────────────────────────────────┘


## 四、代码映射（代码在哪）

| 模块 | 代码位置 | 核心函数 | 在统一 Agent 中的调用方式 |
|------|------|------|------|
| 预处理-Query Rewrite | `app/services/query_rewrite.py` | `rewrite_query(query, history)` | 步骤1：`get_query_rewrite()` 包装调用，做指代消解 |
| 预处理-意图路由 | `app/services/intent_router.py` | `route_intent(question)` | 步骤1：Rewrite 完成后调用，识别意图类别 |
| 工具决策（ReAct 多步） | `app/services/agent.py` | `check_and_call_tool(message, mcp_client)` | 步骤2：LLM 多轮决策 + ThreadPoolExecutor 并行执行同轮 tool_calls |
| MCP 客户端 | `app/services/agent.py` → `MCPClient` | `get_tools_as_fc_format()` / `call_tool()` | 步骤2内部：仅从 MCP 获取工具，无本地兜底 |
| 本地工具定义 | `app/services/tools.py` | `WEATHER_TOOL` / `get_weather()` | 仅旧版 agent_chat 使用，unified_agent 不走此路径 |
| 混合检索 | `app/services/rag.py` | `hybrid_search_rrf(query, top_k)` | 步骤3：`rag_and_rerank()` 包装，始终执行（RRF 排名） |
| Rerank 精排 | `app/services/rerank.py` | `rerank(query, docs, top_k)` | 步骤3：`rag_and_rerank()` 内部，三级降级精排 |
| 合并上下文 | `app/services/agent.py` | `_format_tool_data()` + `insert(0, ...)` | 步骤4：工具数据格式化后插到 RAG 文档最前面 |
| 生成回答 | `app/services/prompt.py` + `agent.py` | `_build_messages()` → `buildPromptTest()` → `generation_agent()` | 步骤5：拼装 messages 后调 LLM 生成 |
| 审核反思 | `app/services/agent.py` | `review_agent(query, answer)` | 步骤6：用原始 user_message 审查初步回答并修正 |
| 对话历史管理 | `app/services/session_manager.py` | `get_history()` / `add_message()` | QR 时读取历史；unified_agent 当前未更新历史 |

---

## 五、Skill 体系设计

一个完整的 Skill 应包含五个要素：

| 要素 | 说明 | 当前天气 Skill 现状 |
|------|------|------|
| 名称 + 描述 | LLM 据此判断是否调用 | ✅ `get_weather` + description |
| 输入 Schema | 参数类型、约束 | ✅ `city: string`（必填）、`date: string`（可选） |
| 执行逻辑 | 真正干活的函数 | ✅ `get_weather()` → 调天气 API |
| 输出 Schema | 返回格式 | ⚠️ 当前返回字符串，建议结构化 `{temperature, humidity, advice}` |
| 错误处理 | 失败兜底策略 | ✅ 超时返回 `(False, "")`，不阻塞主流程 |

**当前问题**：Schema 定义（`WEATHER_TOOL`）和函数实现（`get_weather()`）散落在 `tools.py` 两处。后续重构方向——将五要素封装为统一的 `Skill` 类，Agent 拿到 Skill 即可自举：知道它能干什么、怎么调、调完返回什么、挂了怎么处理。

---

## 六、后续优化点

| # | 优化项 | 当前状态 | 计划 |
|---|------|------|------|
| 1 | MCP 本地兜底 | `_get_available_tools` 仅从 MCP 取 | MCP 工具列表为空时回退 `AVAILABLE_TOOLS` |
| 2 | 对话历史持久化 | unified_agent 当前未更新 session | 回答后调用 `session_manager.add_message()` 写回 |
| 3 | Skill 统一封装 | Schema + 函数散落在 tools.py | 引入 `Skill` 基类，五要素一体化注册 |
| 4 | 多 Agent 编排 | 当前硬编码流水线 | 引入 LangGraph 状态机，支持动态路由和并行 |
| 5 | 流式输出 | unified_agent 仅支持非流式 | 接入 SSE，实现逐 token 推送 |
| 6 | Agent 可观测性 | 无结构化日志 | 每步 Thought/Action/Observation 落库，支持回放和调试 |