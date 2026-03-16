# Token 与上下文

## 2.1 Token 是什么

**Token** 是 LLM 处理文本的**基本单位**。LLM 并不直接处理字符或单词，而是将文本转换为 token 序列进行处理。

### Token 的划分方式

```
"Hello World" → ["Hello", " World"] (2 tokens)

"今天天气很好" → ["今天", "天气", "很好"] (3 tokens)

"AI Agent" → ["AI", " Agent"] (2 tokens)
```

**注意**：
- 英文通常按单词或子词划分
- 中文通常按字或词划分
- 同一个词在不同模型中可能划分方式不同

### Token 数量参考

| 内容类型 | Token 数量 |
|----------|-----------|
| 1 个英文单词 | ~1.3 tokens |
| 1 个中文字 | ~1.5 tokens |
| 1 个英文句子 | ~5-10 tokens |
| 1 个中文句子 | ~8-20 tokens |
| 1000 行代码 | ~15k-25k tokens |
| 一本长篇小说 | ~100k-300k tokens |

## 2.2 Tokenization（分词）

Tokenization 是将文本转换为 token 序列的过程。不同的模型使用不同的分词算法。

### 常见分词算法

#### BPE (Byte Pair Encoding)

**原理**：频繁出现的字节对合并为新的 token

```
初始词汇表：所有 ASCII 字符
↓
统计出现频率最高的字节对
↓
合并高频字节对为新 token
↓
重复直到达到目标词汇量
```

**优点**：
- 词汇量可控
- 能够处理任意文本
- 兼顾词级和字符级的粒度

**代表模型**：GPT-2、GPT-3、GPT-4

#### WordPiece

**原理**：基于语言学的子词划分，最大化词表覆盖

```
构建词表：
1. 初始化基础字符
2. 学习合并规则（基于可能性最大化）
3. 优先保留有意义的子词
```

**优点**：
- 保留完整单词的可能性高
- 适合多语言处理

**代表模型**：BERT、Electra

#### SentencePiece

**原理**：无需预处理，直接在原始文本上训练

```
特点：
- 不假设输入是空格分隔的
- 统一处理所有 Unicode 字符
- 支持 BPE 和 Unigram 语言模型
```

**优点**：
- 适用于任意语言
- 可复现性强

**代表模型**：LLaMA、PaLM

### TikToken 分词器

TikToken 是 OpenAI 使用的快速分词器，基于 BPE 算法。

```python
# 使用 TikToken 进行分词
import tiktoken

# GPT-4 使用 cl100k_base
enc = tiktoken.get_encoding("cl100k_base")

# 分词
tokens = enc.encode("Hello, world!")
print(tokens)  # [9906, 11, 1917, 0]

# 解码
text = enc.decode(tokens)
print(text)  # "Hello, world!"
```

## 2.3 词表（Vocabulary）

词表是分词器能够识别的所有 token 的集合。

| 模型 | 词表大小 |
|------|----------|
| GPT-2 | 50,257 |
| GPT-3 | 50,257 |
| GPT-4 | ~100,000+ |
| LLaMA | 32,000 |
| LLaMA 2 | 32,000 |
| LLaMA 3 | 128,000 |

### 特殊 Token

| Token | 作用 |
|-------|------|
| `<\|pad\|>` | 填充 token |
| `<\|bos\|>` | 文本开始 (Beginning of Sequence) |
| `<\|eos\|>` | 文本结束 (End of Sequence) |
| `<\|unk\|>` | 未知 token |
| `<\|sep\|>` | 分隔符 |
| `<\|system\|>` | 系统消息标记 |

## 2.4 上下文窗口（Context Window）

上下文窗口是 LLM 一次能够处理的最大 token 数量。

### 常见模型的上下文窗口

| 模型 | 上下文窗口 |
|------|-----------|
| GPT-3.5 | 4k / 16k tokens |
| GPT-4 | 8k / 32k / 128k tokens |
| Claude 2 | 100k tokens |
| Claude 3 | 200k tokens |
| Gemini 1.5 | 1M tokens |
| LLaMA | 32k tokens |
| LLaMA 2 | 32k / 128k tokens |
| LLaMA 3 | 8k tokens |

### 为什么需要上下文窗口

1. **计算资源限制**：Attention 机制的计算复杂度是 O(n²)，n 越大计算量越大
2. **显存限制**：KV Cache 需要存储所有历史 token 的注意力键值
3. **内存成本**：长上下文需要大量显存

### 扩展上下文窗口的技术

#### 位置编码扩展

**RoPE (Rotary Position Embedding)**：
- 旋转位置编码，支持长上下文外推
- 通过旋转矩阵实现位置信息编码

**ALiBi (Attention with Linear Biases)**：
- 在注意力分数上添加线性偏置
- 使模型能够处理超出训练长度的序列

#### 推理优化技术

**Flash Attention**：
- IO 优化的注意力计算
- 减少显存占用，加速计算

**KV Cache 优化**：
- 缓存已计算的 Key 和 Value
- 避免重复计算

**分块计算（Chunked Attention）**：
- 将长序列分块处理
- 减少峰值显存使用

### 上下文窗口的实际影响

```
上下文窗口限制：
- 无法一次处理超长文档
- 长对话可能丢失早期信息
- RAG 需要分块处理文档
```

## 2.5 Token 数量估算

### 经验公式

```
英文：token ≈ word × 1.3
中文：token ≈ character × 1.5
代码：token ≈ line × 20 (取决于语言和复杂度)
```

### API 计费

```
成本 = (输入 token 数 + 输出 token 数) × 单价

示例 (GPT-4):
- 输入：$10 / 1M tokens
- 输出：$30 / 1M tokens
```

## 常见问题

### Q: 为什么同一个词在不同模型中 token 数不同？

不同模型使用不同的分词器，词汇表大小和分词算法不同，导致同一个词的 token 划分方式不同。

### Q: 如何估算一段文本的 token 数量？

可以使用分词器的 `encode()` 方法获取精确数量，或使用经验公式估算。

### Q: 上下文窗口满了会怎样？

超出上下文窗口的内容会被截断，模型无法看到更早的信息。这可能导致：
- 长对话中丢失早期上下文
- 长文档处理不完整
- 需要使用滑动窗口或记忆压缩技术