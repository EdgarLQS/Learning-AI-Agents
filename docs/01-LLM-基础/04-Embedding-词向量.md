# Embedding 词向量

## 4.1 概念定义

**Embedding（词向量/嵌入）** 是将文本（词、句子、文档）转换为密集向量表示的技术。

```
"猫" → [0.2, -0.5, 0.8, 0.1, ...] (768维或更高)
"狗" → [0.1, -0.4, 0.9, 0.2, ...]
"汽车" → [-0.6, 0.3, -0.2, -0.5, ...]
```

### 为什么需要 Embedding

- **计算机无法直接处理文本**：计算机只能处理数值
- **语义表示**：将语义信息编码到向量空间
- **数学运算**：在向量空间中进行语义计算

## 4.2 词向量发展历程

### 4.2.1 One-Hot 编码

最简单的方法，每个词一个向量：

```
"猫": [1, 0, 0, 0, ...]
"狗": [0, 1, 0, 0, ...]
"猫"和"狗"的内积为0，无法表示相似性
```

**问题**：
- 维度爆炸（词汇表大小）
- 无法表示语义相似性
- 稀疏表示效率低

### 4.2.2 Word2Vec

Google 2013 年提出，通过神经网络学习词向量。

#### CBOW (Continuous Bag of Words)

```
上下文: ["我", "喜欢", "学习"]
目标词: "AI"

预测: P(AI | 我, 喜欢, 学习)
```

#### Skip-Gram

```
目标词: "AI"
上下文: ["我", "喜欢", "学习"]

预测: P(我|AI), P(喜欢|AI), P(学习|AI)
```

#### 特点

- **分布式表示**：语义相似的词向量相近
- **低维稠密**：通常 100-300 维
- **语义运算**：`国王 - 男人 + 女人 ≈ 女王`

### 4.2.3 GloVe (Global Vectors)

结合全局统计信息和局部上下文：

```
目标函数：最小化共现矩阵与词向量点积的差异

优点：
- 训练更快
- 效果更好
- 利用全局信息
```

### 4.2.4 上下文相关词向量（ELMo）

2018 年提出，解决一词多义问题：

```
"银行" 在不同上下文中的向量不同：

1. "我去银行存钱" → [0.3, -0.1, 0.5, ...]
2. "河边的银行" → [-0.2, 0.4, -0.3, ...]
```

**创新**：
- 双向 LSTM
- 动态上下文表示
- 预训练 + 下游任务微调

### 4.2.5 Transformer Embedding

现代 LLM 使用 Transformer 学习词向量：

- BERT：双向 Transformer Encoder
- GPT：单向 Transformer Decoder
- 直接作为模型的一部分端到端训练

## 4.3 词向量的特性

### 4.3.1 语义相似性

```
语义相似的词，向量距离近

向量空间示例：
          猫
          |
    狗 ---+
          |
         宠物

汽车与猫的距离 > 狗与猫的距离
```

### 4.3.2 语义运算

```
king - man + woman ≈ queen
Paris - France + Italy ≈ Rome
walk - walked ≈ swim - swam
```

### 4.3.3 聚类特性

```
同类型词在向量空间中形成聚类：
- 动物: 猫、狗、鱼、鸟...
- 水果: 苹果、香蕉、橙子...
- 动作: 跑、跳、游泳...
```

## 4.4 向量相似度计算

### 4.4.1 余弦相似度（Cosine Similarity）

最常用的相似度度量：

```
cosine(A, B) = (A · B) / (||A|| × ||B||)

范围：[-1, 1]
- 1: 方向相同
- 0: 正交（无关）
- -1: 方向相反
```

```python
import numpy as np

def cosine_similarity(a, b):
    dot_product = np.dot(a, b)
    norm_a = np.linalg.norm(a)
    norm_b = np.linalg.norm(b)
    return dot_product / (norm_a * norm_b)
```

### 4.4.2 欧氏距离

```
distance(A, B) = ||A - B||

特点：
- 受向量 magnitude 影响
- 适合归一化后的向量
```

### 4.4.3 点积（Dot Product）

```
similarity(A, B) = A · B

特点：
- 计算简单
- 受 magnitude 影响
- 归一化后等价于余弦相似度
```

## 4.5 词向量维度

| 模型 | 向量维度 |
|------|----------|
| Word2Vec | 100-300 |
| GloVe | 50-300 |
| BERT | 768 / 1024 |
| GPT-2 | 1024 |
| GPT-3 | 12288 |
| LLaMA | 4096-8192 |

### 维度选择

- **过低**：表达能力不足，无法捕捉复杂语义
- **过高**：计算成本增加，可能过拟合
- **经验**：通常选择 256-4096

## 4.6 Embedding 在 LLM 中的应用

### 4.6.1 输入 Embedding

```python
# 简化示意
token_ids = [101, 2054, 2003, 1037, 2143, 102]
embedding_layer = model.embed_tokens
input_embeddings = embedding_layer(token_ids)
# shape: (1, 6, 4096)
```

### 4.6.2 输出 Logits

```python
# 最后一层隐藏状态 -> 词汇表概率
logits = lm_head(hidden_states)
# shape: (batch, seq_len, vocab_size)
```

### 4.6.3 语义搜索

```python
# 文档向量化
doc_embedding = model.encode("今天天气很好")

# 查询向量化
query_embedding = model.encode("天气怎么样")

# 相似度计算
similarity = cosine_similarity(query_embedding, doc_embedding)
```

## 4.7 常见的 Embedding API

| 服务 | 模型 | 维度 | 免费额度 |
|------|------|------|----------|
| OpenAI | text-embedding-3-small | 1536 | $0.02/1M |
| OpenAI | text-embedding-3-large | 3072 | $0.13/1M |
| HuggingFace | sentence-transformers | 384-1024 | 免费 |
| Cohere | embed-multilingual | 1024 | 免费 |

## 常见问题

### Q: 为什么词向量能表示语义？

通过在大规模文本上训练，模型学习到语义相似的词在上下文中经常一起出现，因此它们的向量表示也会相近。

### Q: 词向量是固定的吗？

- **静态词向量**（Word2Vec、GloVe）：训练后固定
- **上下文相关词向量**（BERT、GPT）：根据输入动态变化
- **端到端训练**：作为 LLM 的一部分一起训练

### Q: 如何评估词向量的质量？

- **语义运算**：king - man + woman ≈ queen
- **相似度任务**：人类标注的相似度与计算相似度对比
- **下游任务**：在具体任务上的表现