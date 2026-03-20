# Transformer 架构

## 3.1 概述

Transformer 是 LLM 的核心架构，由 Google 在 2017 年论文 **"Attention Is All You Need"** 中提出。

### 论文信息

```
标题：Attention Is All You Need
作者：Vaswani et al.
年份：2017
机构：Google Brain, Google Research, University of Toronto
```

### Transformer 的核心创新

- **纯 Attention 机制**：完全摒弃 RNN/LSTM，仅使用 Attention
- **自注意力（Self-Attention）**：让每个位置关注所有位置
- **并行计算**：解决 RNN 序列依赖问题，大幅加速训练

### 核心概念对照表 (Core Concepts)

| Concept / 概念 | Description / 描述 |
|---------------|-------------------|
| Self-Attention / 自注意力 | Allows the model to weigh the importance of different words in a sentence relative to a specific word. / 允许模型根据特定单词来衡量句子中不同单词的重要性。 |
| Positional Encoding / 位置编码 | Injects information about the order of words since the model processes everything in parallel. / 由于模型是并行处理所有内容的，因此需要注入有关单词顺序的信息。 |
| Multi-Head Attention / 多头注意力 | Enables the model to focus on different parts of the sequence at the same time for better context. / 使模型能够同时关注序列的不同部分，以获得更好的上下文。 |

## 3.2 整体架构

```
┌─────────────────────────────────────────────────┐
│                    输入 Embedding                 │
│              (Input Embedding + Positional Encoding)      │
└─────────────────────┬───────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│              Transformer Encoder                 │
│  ┌─────────────────────────────────────────┐    │
│  │           Multi-Head Self-Attention      │    │
│  └──────────────────┬──────────────────────┘    │
│                     ↓                           │
│  ┌─────────────────────────────────────────┐    │
│  │            Add & LayerNorm               │    │
│  └──────────────────┬──────────────────────┘    │
│                     ↓                           │
│  ┌─────────────────────────────────────────┐    │
│  │              Feed Forward                │    │
│  └──────────────────┬──────────────────────┘    │
│                     ↓                           │
│  ┌─────────────────────────────────────────┐    │
│  │            Add & LayerNorm               │    │
│  └─────────────────────────────────────────┘    │
│                  × N 层                          │
└─────────────────────┬───────────────────────────┘
                      ↓
                 输出概率
```

### 编码器工作流程 (Encoder Workflow)

| Step / 步骤 | Component / 组件 | Purpose / 目的 |
|-----------|-----------------|--------------|
| 1 | Input Embedding + Positional Encoding | Combining meaning with word order. 🧩 / 结合含义与词序 |
| 2 | Multi-Head Self-Attention | Looking at the sentence from different "perspectives" (grammar, logic, etc.). 🔍 / 从不同"视角"（语法、逻辑等）查看句子 |
| 3 | Add & Norm / 残差连接与归一化 | Keeping the training stable and preventing information loss. ⚖️ / 保持训练稳定性并防止信息丢失 |
| 4 | Feed-Forward Network / 前馈神经网络 | Processing each word's information further in isolation. ⚙️ / 独立处理每个单词的信息 |

### 编码器 - 解码器结构

Transformer 包含两个主要部分：

#### Encoder（编码器）

- 处理输入序列
- 提取特征表示
- 典型应用：BERT

#### Decoder（解码器）

- 生成输出序列
- 自回归生成
- 典型应用：GPT 系列

### 解码器三要素 (Decoder Key Features)

| Feature / 特性 | Description / 描述 |
|--------------|-----------------|
| Masked Self-Attention / 掩码自注意力 | Hides future words so the model only looks at what it has written so far. 🚫 / 隐藏未来单词，使模型只关注到目前为止已写的内容 |
| Encoder-Decoder Attention / 编码器 - 解码器注意力 | This is the "bridge." The Decoder looks back at the Encoder's "map" to find relevant info from the source. 🌉 / 这是"桥梁"。解码器回看编码器的"地图"以从源中找到相关信息 |
| Linear & Softmax / 线性层与 Softmax | Turns the internal math into a probability for every possible word in the dictionary. 📊 / 将内部数学转换为字典中每个可能单词的概率 |

### Encoder-Only vs Decoder-Only

| 类型 | 结构 | 特点 | 代表模型 |
|------|------|------|----------|
| Encoder-Only | 双向 Attention | 理解能力强，适合分类 | BERT |
| Decoder-Only | 单向 Attention | 生成能力强 | GPT 系列 |
| Encoder-Decoder | 编码器 + 解码器 | 翻译、摘要 | T5, BART |

## 3.3 各组件详解

### 3.3.1 位置编码（Positional Encoding）

由于 Attention 本身不包含位置信息，需要通过位置编码注入位置信号。

#### 位置编码的核心思想

| Part / 部分 | Description / 描述 |
|-----------|-----------------|
| Word Embedding / 词嵌入 | The meaning of the word (e.g., "River"). / 单词的含义（如"River"） |
| Positional Vector / 位置向量 | The "page number" (e.g., "Position 4"). / "页码"（如"位置 4"） |
| The Result / 结果 | Meaning + Position = A word that knows where it stands. / 含义 + 位置 = 一个知道自己位置的单词 |

#### 绝对位置编码

```
PE(pos, 2i) = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

其中：
- `pos`：位置索引
- `i`：维度索引
- `d_model`：模型维度

#### 相对位置编码

- 编码 token 之间的相对距离
- 更适合处理长序列
- 代表：ALiBi、RoPE

#### RoPE（旋转位置编码）

```
旋转矩阵：
[[cos(θ), -sin(θ)],
 [sin(θ),  cos(θ)]]

通过旋转操作将位置信息编码到向量中
```

**优点**：
- 支持长上下文外推
- 计算高效
- 无需额外参数

### 3.3.2 自注意力层（Self-Attention）

#### Q, K, V 详解 (Query, Key, Value Explanation)

**核心类比**:
```
Query (Q) / 查询 (Q): The "search term" the word uses to look at other words.
                    单词用来查找其他单词的"搜索词"。

Key (K) / 键 (K):     The "label" that describes what a word has to offer.
                    描述一个单词能提供什么的"标签"。

Value (V) / 值 (V):   The actual content or "meaning" of the word.
                    单词的实际内容或"含义"。
```

#### 注意力流程三步骤 (Attention Flow)

1. **Match / 匹配**: Q is multiplied by K to get a score. / Q 与 K 相乘以获得注意力得分。
2. **Normalize / 归一化**: The scores are turned into percentages (probabilities) using Softmax. / 使用 Softmax 函数将得分转换为百分比（概率）。
3. **Combine / 组合**: The model takes the Values (V) and weighs them by those percentages. / 模型获取值 (V) 并按这些百分比进行加权。

#### 数学公式

```
输入：X (batch_size, seq_len, d_model)

1. 线性投影生成 Q, K, V
   Q = XW_Q, K = XW_K, V = XW_V

2. 计算注意力分数 (Scaled Dot-Product Attention)
   Attention(Q, K, V) = softmax(QK^T / √d_k)V

   公式步骤:
   - QK^T: Query 与 Key 的转置相乘得到注意力分数
   - / √d_k: 除以 Key 向量维度的平方根进行缩放
   - softmax(): 将分数归一化为概率分布
   - × V: 用概率加权 Value 向量

3. 输出：(batch_size, seq_len, d_model)
```

### 3.3.3 多头注意力（Multi-Head Attention）

```
并行执行多次 Attention：
Head_i = Attention(QW_Qi, KW_Ki, VW_Vi)

输出 = Concat(Head_1, ..., Head_h)W_O

参数：
- h: 头数（通常 16-32）
- d_k: 每个头的维度
```

#### Single-Head vs Multi-Head 对比

| Feature / 特性 | Single-Head / 单头 | Multi-Head / 多头 |
|--------------|------------------|-----------------|
| Focus / 焦点 | One relationship at a time | Multiple relationships simultaneously |
| Analogy / 类比 | One spotlight 🔦 | A whole lighting rig 🏗️ |
| Result / 结果 | Might miss secondary details | Captures location, state, and grammar at once |

### 3.3.4 前馈网络（Feed Forward Network）

通常为两层全连接网络：

```
FFN(x) = GELU(xW_1 + b_1)W_2 + b_2

典型配置：
- d_model = 4096
- d_ff = 16384 (4x d_model)
```

### 3.3.5 残差连接与 Layer Normalization

```
LayerNorm(x + Sublayer(x))

残差连接：
- 缓解梯度消失
- 稳定训练

Layer Normalization：
- 稳定训练
- 加速收敛
- 位置：每个子层之后
```

## 3.4 模型层数

| 模型 | 层数 | 参数量 |
|------|------|--------|
| BERT-Base | 12 | 110M |
| BERT-Large | 24 | 340M |
| GPT-2 | 48 | 1.5B |
| GPT-3 175B | 96 | 175B |
| GPT-4 | ~120 | ~1.8T |
| LLaMA 70B | 80 | 70B |
| Claude | ~100+ | 未公开 |

## 3.5 训练目标

### Encoder-Only (BERT)

```
MLM (Masked Language Model)：
- 随机 mask 15% 的 token
- 预测被 mask 的 token

NSP (Next Sentence Prediction)：
- 判断两个句子是否相邻
```

### Decoder-Only (GPT)

```
Next Token Prediction：
- 根据前文预测下一个 token
- 训练目标：最大化对数似然
```

### Encoder-Decoder (T5)

```
Text-to-Text：
- 所有任务统一为文本到文本格式
- 翻译、摘要、问答等
```

## 3.6 从 Transformer 到 LLMs (From Transformer to LLMs)

### LLM 架构类型对比

| Model Type / 模型类型 | Architecture / 架构 | Focus / 重点 | Examples / 例子 |
|---------------------|-------------------|-------------|----------------|
| **Encoder-Only** / 仅编码器 | Understanding & Classifying / 理解与分类 | Understanding the relationship between all words at once. / 同时理解所有单词之间的关系 | **BERT** |
| **Decoder-Only** / 仅解码器 | Generation & Prediction / 生成与预测 | Predicting the very next word in a sequence. / 预测序列中的下一个单词 | **GPT-4, Gemini** |
| **Encoder-Decoder** / 编码器 - 解码器 | Translation & Summarization / 翻译与摘要 | Mapping one sequence directly to another. / 将一个序列直接映射到另一个序列 | **T5, BART** |

### 为什么 Decoder-Only (GPT 风格) 胜出？

> While Encoders are great at understanding, **Decoder-only** models turned out to be incredibly "scalable." By simply training them to predict the next word on massive amounts of internet text, they developed emergent abilities like reasoning, coding, and creative writing.
>
> 虽然编码器擅长理解，但事实证明**仅解码器**模型具有惊人的"可扩展性"。通过简单地训练它们在海量互联网文本上预测下一个单词，它们发展出了推理、编程和创意写作等涌现能力。

## 3.7 计算复杂度

### Self-Attention

```
时间复杂度：O(n² × d)
空间复杂度：O(n² × d)

其中：
- n: 序列长度
- d: 模型维度
```

### 前馈网络

```
时间复杂度：O(n × d × d_ff)
空间复杂度：O(n × d × d_ff)
```

## 常见问题

### Q: Transformer 为什么能处理长序列？

通过 Self-Attention，每个 token 可以直接与任意位置交互，但代价是 O(n²) 的计算复杂度。实际应用中需要通过各种优化技术来加速。

### Q: 为什么层数越多需要更多显存？

每层都需要保存：
- 注意力权重
- 中间激活值
- 梯度信息

层数增加导致这些中间状态线性增长。

### Q: Encoder 和 Decoder 有什么区别？

- **Encoder**：使用双向 Attention，可以看到完整输入
- **Decoder**：使用单向（掩码）Attention，只能看到前缀
- **Encoder-Decoder**：Encoder 处理输入，Decoder 基于 Encoder 输出生成

## 学习总结 (Learning Summary)

| 主题 | 核心类比 | 表情符号 |
|------|---------|---------|
| **Self-Attention (Q, K, V)** / 自注意力机制 | How the model uses "spotlights" to find meaning / 模型如何利用"聚光灯"寻找含义 | 🔍 |
| **Positional Encoding** / 位置编码 | How the model understands word order using "page numbers" / 模型如何利用"页码"理解词序 | 🔢 |
| **Encoder-Decoder Structure** / 编码器 - 解码器结构 | How the two halves collaborate for tasks like translation / 这两个部分如何协作完成翻译等任务 | 🏗️ |
| **The Rise of LLMs** / 大语言模型的兴起 | How these ideas evolved into models like GPT and BERT / 这些想法如何演变成 GPT 和 BERT 等模型 | 🚀 |
