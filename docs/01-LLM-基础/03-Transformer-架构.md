# Transformer 架构

## 3.1 概述

Transformer 是 LLM 的核心架构，由 Google 在 2017 年论文 **"Attention Is All You Need"** 中提出。

### 论文信息

```
标题: Attention Is All You Need
作者: Vaswani et al.
年份: 2017
机构: Google Brain, Google Research, University of Toronto
```

### Transformer 的核心创新

- **纯 Attention 机制**：完全摒弃 RNN/LSTM，仅使用 Attention
- **自注意力（Self-Attention）**：让每个位置关注所有位置
- **并行计算**：解决 RNN 序列依赖问题，大幅加速训练

## 3.2 整体架构

```
┌─────────────────────────────────────────────────┐
│                    输入Embedding                 │
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

### 编码器-解码器结构

Transformer 包含两个主要部分：

#### Encoder（编码器）

- 处理输入序列
- 提取特征表示
- 典型应用：BERT

#### Decoder（解码器）

- 生成输出序列
- 自回归生成
- 典型应用：GPT 系列

### Encoder-Only vs Decoder-Only

| 类型 | 结构 | 特点 | 代表模型 |
|------|------|------|----------|
| Encoder-Only | 双向 Attention | 理解能力强，适合分类 | BERT |
| Decoder-Only | 单向 Attention | 生成能力强 | GPT 系列 |
| Encoder-Decoder | 编码器+解码器 | 翻译、摘要 | T5, BART |

## 3.3 各组件详解

### 3.3.1 位置编码（Positional Encoding）

由于 Attention 本身不包含位置信息，需要通过位置编码注入位置信号。

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

- 编码token之间的相对距离
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

```
输入: X (batch_size, seq_len, d_model)

1. 线性投影生成 Q, K, V
   Q = XW_Q, K = XW_K, V = XW_V

2. 计算注意力分数
   Attention(Q, K, V) = softmax(QK^T / √d_k)V

3. 输出: (batch_size, seq_len, d_model)
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

## 3.6 计算复杂度

### Self-Attention

```
时间复杂度: O(n² × d)
空间复杂度: O(n² × d)

其中：
- n: 序列长度
- d: 模型维度
```

### 前馈网络

```
时间复杂度: O(n × d × d_ff)
空间复杂度: O(n × d × d_ff)
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