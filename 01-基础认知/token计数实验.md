# Token 计数实验

> 📅 学习日期：2026-08-08
> 🔗 关联面试题：提示词工程专题 Token 相关题目

> 待补充：中英文 token 计数差异、不同模型 tokenizer 对比实验记录


## 1. Token 是什么

Token 是 LLM 处理和计费的基本单位。它不是“字”，也不是“词”，而是介于两者之间的“语义片段”。

**用你已经熟悉的类比**：就像前端的 CSS 像素——不是物理像素，而是一个逻辑单位。Token 就是 LLM 世界的“逻辑像素”。

**中文 vs 英文的 Token 差异**：
- 英文：1 个词 ≈ 1-2 个 Token（"Hello" = 1 token，"beautiful" = 2 tokens）
- 中文：1 个字 ≈ 1-2 个 Token（“宝宝” ≈ 2 tokens，“发烧” ≈ 2 tokens）

## 2. 怎么知道一次请求消耗了多少 Token

每次调用 DeepSeek API，返回结果中都会包含一个 `usage` 字段：

```json
{
  "usage": {
    "prompt_tokens": 480,       // 输入消耗的 Token
    "completion_tokens": 55,     // 输出消耗的 Token
    "total_tokens": 535          // 总共消耗的 Token
  }
}
```

**三个字段的含义：**

| 字段 | 含义 | 包含什么 |
|------|------|---------|
| `prompt_tokens` | 输入 Token | System Prompt + 检索到的文档 + 用户问题 |
| `completion_tokens` | 输出 Token | LLM 生成的回答 |
| `total_tokens` | 总计 | prompt + completion |

## 3. Token 是怎么算钱的

DeepSeek 官方参考价格：

| 计费项 | 价格 |
|--------|------|
| 输入 Token | ¥1 / 百万 Token |
| 输出 Token | ¥2 / 百万 Token |

**举个例子：**一次 RAG 问答消耗了 500 prompt tokens + 200 completion tokens：

```text
费用 = (500 × 1 + 200 × 2) ÷ 1,000,000 
     = 900 ÷ 1,000,000 
     = ¥0.0009 元
```

也就是说，一次 RAG 问答的成本不到 1 分钱。DeepSeek 的价格非常便宜，在开发和小规模使用阶段，Token 成本几乎可以忽略不计。

### 3.1 中英文计数差异

根源在于模型底层的 Tokenizer（分词器） 实现不同。

- 英文优化：大多数 Tokenizer 的训练数据以英文为主，因此对英文单词和词根的组合非常高效。

- 中文挑战：中文不是由字母组成，而是由成千上万的汉字构成。Tokenizer 在处理中文时，会将它切分成更细粒度的“子词”或“单字”。这导致表达同样信息量时，中文往往需要更多 Token。

- 模型优化：以 DeepSeek 为代表的国产模型，其 Tokenizer 针对中文进行了专门优化，能够更好地识别常见中文词汇，从而减少 Token 消耗。

## 4. 我的项目里怎么获取 Token 用量

### 4.1 代码位置

`app/core/cost_tracker.py`

## 4.2 核心逻辑

```python
# 从 API 返回中提取 Token 数
if hasattr(response, 'usage') and response.usage:
    prompt_tokens = response.usage.prompt_tokens      # 输入 Token
    completion_tokens = response.usage.completion_tokens  # 输出 Token
    
    # 记录到成本追踪器
    cost_tracker.record(prompt_tokens, completion_tokens)
```

### 4.3 实际效果

我的项目中有一个` /api/admin/cost` 接口，可以实时查看当日的 Token 消耗：
```json
{
  "prompt_tokens": 12500,
  "completion_tokens": 3400,
  "total_tokens": 15900,
  "estimated_cost_yuan": 0.0193
}
```
花了不到 2 分钱。这说明在开发和小规模使用阶段，Token 费用根本不是瓶颈——真正需要关注的是检索质量、生成准确性和系统架构，成本问题等规模化之后再考虑。

## 5. 为什么 Token 计数很重要

| 原因 | 说明 |
|------|------|
| 成本意识 | 知道每次请求花多少钱，才能判断什么时候需要优化 |
| 上下文窗口管理 | LLM 有 Token 上限（如 128K），超出的内容会被截断 |
| Prompt 优化 | 知道哪些部分吃 Token 最多（通常是检索文档），才能做针对性优化 |
| 模型选型 | 不同模型价格差异大，Token 数据是选型的重要依据 |


## 6. 我的发现

- RAG 请求中，检索文档是最大的 Token 消耗源——如果一次检索返回 5 篇文档，每篇 500 字，光是上下文就占了 3000+ Token。

- 这意味着我做的混合检索和 Rerank 精排（只保留 top-5）不仅是精度优化，也是成本优化——减少注入 LLM 的文档数量，直接降低 Token 消耗。

- DeepSeek 的 API 价格极低，当前项目完全不需要担心成本，但养成了 Token 计数的习惯，未来扩展时能立刻知道哪部分开销最大。

## 7. 项目关联

- Token 追踪：`app/core/cost_tracker.py`

- LLM 调用集成：`app/services/llm.py`

- 成本查询接口：`app/main.py → /api/admin/cost`