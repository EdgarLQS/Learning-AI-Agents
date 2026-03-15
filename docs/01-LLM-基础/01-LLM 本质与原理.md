# LLM 基础

## 01-LLM 本质与原理

### LLM 本质

LLM (Large Language Model) 本质是**概率预测模型**：

```
LLM 本质 = 输入文本 → 概率计算 → 预测下一个 token
```

**核心公式**：
```
P(next_token | context) → 输出概率分布 → 选择 token
```

### 例子

输入：`"今天天气"`

模型预测：
- `"很好"` (35%)
- `"不错"` (28%)
- `"晴朗"` (15%)
- ...

## 02-Token 与上下文

### Token 是什么

Token 是 LLM 处理文本的基本单位。

**例子**：
```
"Hello World" → ["Hello", " World"] (2 tokens)

"代码" → ["代", "码"] (中文通常按字拆分)

函数定义：
def hello():
    return "world"
→ 约 15-20 tokens
```

### 常见模型上下文窗口

| 模型 | 上下文窗口 |
|------|-----------|
| GPT-4 | 128k tokens |
| Claude | 200k tokens |
| Gemini | 1M tokens |
| LLaMA | 32k tokens |

### Token 数量参考

| 内容类型 | Token 数量 |
|----------|-----------|
| 1 个英文单词 | ~1.3 tokens |
| 1 个中文字 | ~1.5 tokens |
| 1000 行代码 | ~15k-25k tokens |
| 一本长篇小说 | ~100k-300k tokens |

## 03-Transformer 架构

### 论文

"Attention Is All You Need" (2017)

### Transformer 结构

```
输入 → Embedding → Transformer Layers → 输出
                    ↓
               Multi-Head Attention
               Feed Forward Network
               Layer Normalization
```

### 模型层数

| 模型 | 层数 |
|------|------|
| GPT-4 | ~96 层 |
| Claude | ~100+ 层 |
| LLaMA | 32-80 层 |

## 04-Embedding 词向量

### 概念

Embedding 是将文本转换为向量表示：

```
"猫" → [0.2, -0.5, 0.8, ...] (768 维或更高)
"狗" → [0.1, -0.4, 0.9, ...]
"汽车" → [-0.6, 0.3, -0.2, ...]
```

### 特性

- 语义相似的词向量距离近
- 支持向量运算：`国王 - 男人 + 女人 ≈ 女王`

## 05-Attention 机制

### 核心公式

```
Attention(Q, K, V) = softmax(QK^T / √d) × V
```

### 作用

Attention 机制让模型能够关注输入中重要的部分。

**例子**：
```
输入："我去了银行取钱，因为需要现金"

当理解"银行"时，Attention 会关注：
- "取钱" (强关联)
- "现金" (强关联)

从而区分"银行"(金融机构) 和"河岸"
```

### Self-Attention 过程

1. 每个词计算 Query(Q)、Key(K)、Value(V)
2. 计算注意力分数：Q × K^T
3. Softmax 归一化
4. 加权求和：attention_scores × V

## 06-LLM 训练过程

### Pretraining (预训练)

**目标**：预测下一个 token

```
输入："今天天气____"
目标："很好"
Loss = CrossEntropy(预测，目标)
```

### 训练数据

| 模型 | 训练数据量 |
|------|-----------|
| GPT-3 | 175B tokens |
| GPT-4 | ~1T tokens |
| LLaMA3 | 70B tokens |

**数据来源**：
- 网页文本
- 书籍
- 代码 (GitHub 等)
- 维基百科

### 训练规模

- GPU 数量：数千到数万
- 训练时间：数周到数月
- 成本：数百万到数亿美元

## 07-推理与采样策略

### 推理过程

```
输入 prompt → 处理 → 输出 token 1 → 输出 token 2 → ... → 完成
```

### 采样策略

| 策略 | 说明 |
|------|------|
| Greedy | 选最大概率 token |
| Top-k | 从前 k 个中随机选 |
| Top-p | 累积概率达到 p 的范围内选 |
| Temperature | 控制随机性 (高=更随机) |

## 08-LLM 能力与局限

### 能力来源

1. **训练数据**：见过的模式越多，能力越强
2. **模型规模**：参数越多，表达能力越强
3. **架构设计**：更好的 Attention 机制

### 能力

- 文本生成
- 代码编写
- 翻译
- 问答
- 推理

### 局限性

1. **幻觉 (Hallucination)**：可能编造信息
2. **上下文限制**：无法处理超长输入
3. **不稳定推理**：复杂逻辑可能出错
4. **知识截止**：训练后不知道新信息

```
LLM 本身 = 被动回答问题

LLM + Agent = 主动执行任务

AI Agent = LLM + Planning + Tools + Memory
```
