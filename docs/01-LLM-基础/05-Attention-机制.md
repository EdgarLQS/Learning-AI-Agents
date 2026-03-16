# Attention 机制

## 5.1 概念引入

**Attention（注意力机制）** 是 Transformer 的核心组件，让模型能够关注输入中最重要的部分。

### 为什么需要 Attention

传统序列模型（RNN/LSTM）的局限：
- 难以捕捉长距离依赖
- 序列化计算效率低
- 梯度消失/爆炸问题

Attention 的优势：
- 直接建立任意位置之间的联系
- 并行计算效率高
- 能够更好地处理长序列

## 5.2 Scaled Dot-Product Attention

### 核心公式

```
Attention(Q, K, V) = softmax(QK^T / √d_k) × V
```

其中：
- `Q` (Query)：查询向量，表示当前位置"想要什么"
- `K` (Key)：键向量，表示每个位置"提供什么信息"
- `V` (Value)：值向量，表示每个位置的"实际内容"
- `d_k`：Key 向量的维度

### 计算流程

```
1. 输入：X (batch, seq_len, d_model)

2. 线性投影
   Q = XW_Q, K = XW_K, V = XW_V
   shape: (batch, seq_len, d_k/d_v)

3. 计算注意力分数
   scores = QK^T / √d_k
   shape: (batch, seq_len, seq_len)

4. Softmax 归一化
   attention_weights = softmax(scores, dim=-1)

5. 加权求和
   output = attention_weights × V
   shape: (batch, seq_len, d_v)
```

### 为什么要除以 √d_k

```
问题：当 d_k 较大时，QK^T 的值可能很大
     导致 Softmax 梯度极小（进入饱和区）

解决：除以 √d_k 进行缩放
     使方差保持为 1，稳定训练

数学证明：
- 假设 Q 和 K 的每个元素均值为 0，方差为 1
- QK^T 的均值为 0，方差为 d_k
- 除以 √d_k 后，方差为 1
```

## 5.3 Multi-Head Attention（多头注意力）

### 原理

并行执行多个 Attention，将结果拼接：

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h)W^O

head_i = Attention(QW_Q^i, KW_K^i, VW_V^i)
```

### 特点

- **多视角学习**：每个头可以关注不同类型的关系
- **并行计算**：各头之间独立，可并行执行
- **可学习参数**：每个头有独立的投影矩阵

### 头数配置

| 模型 | 头数 | 每头维度 |
|------|------|----------|
| BERT-Base | 12 | 64 |
| BERT-Large | 16 | 64 |
| GPT-3 | 96 | 128 |
| LLaMA | 32 | 128 |

### 不同类型的 Attention

#### Self-Attention（自注意力）

```
Q, K, V 都来自同一个输入

应用：Transformer 编码器层
```

#### Masked Self-Attention（掩码自注意力）

```
在 Self-Attention 基础上掩码未来位置

应用：Transformer 解码器层
```

#### Encoder-Decoder Attention（编码器-解码器注意力）

```
Q 来自解码器，K 和 V 来自编码器

应用：Seq2Seq 模型的解码器
```

## 5.4 注意力可视化示例

### 例子：理解"银行"

```
输入："我去了银行取钱，因为需要现金"

当理解"银行"时，Attention 关注：
┌────────────────────────────────┐
│  我      去了    银行    取钱  │
│ [0.05]  [0.08]  [目标] [0.45] │
│                              │
│  因为    需要    现金          │
│ [0.12]  [0.10]  [0.20]        │
└────────────────────────────────┘

解释：
- "取钱"和"现金"与"银行"高度相关
- 模型由此区分"银行"(金融机构) 和 "河岸"
```

### 多头注意力的不同角色

```
头1 - 语法关系：关注动词与名词
头2 - 语义关联：关注同义词/反义词
头3 - 位置关系：关注相邻词
头4 - 指代消解：关注代词指代
```

## 5.5 计算复杂度分析

### Self-Attention

```
时间复杂度: O(n² × d)
- n: 序列长度
- d: 模型维度

空间复杂度: O(n² × d)
- 需要存储注意力矩阵
```

### 相比 RNN

| 模型 | 时间复杂度 | 空间复杂度 | 最大依赖距离 |
|------|-----------|-----------|-------------|
| RNN | O(n × d) | O(d) | O(n) |
| CNN | O(n × k × d) | O(k × d) | O(k) |
| Self-Attention | O(n² × d) | O(n² × d) | O(1) |

### 优化方法

#### 稀疏注意力（Sparse Attention）

```
只关注部分位置，而非全部
- 局部窗口
- 随机采样
- 固定步长
```

#### 线性注意力（Linear Attention）

```
将 O(n²) 降为 O(n)
- 使用核函数近似
- 近似方法精度损失
```

## 5.6 实际实现

### PyTorch 简明实现

```python
import torch
import torch.nn.functional as F
import math

def scaled_dot_product_attention(Q, K, V, mask=None):
    d_k = Q.size(-1)
    scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)

    if mask is not None:
        scores = scores.masked_fill(mask == 0, -1e9)

    attention_weights = F.softmax(scores, dim=-1)
    return torch.matmul(attention_weights, V)

# 使用
output = scaled_dot_product_attention(Q, K, V, mask)
```

### 带 Mask 的自注意力

```python
def causal_mask(seq_len):
    """创建下三角掩码，防止看到未来位置"""
    mask = torch.tril(torch.ones(seq_len, seq_len))
    return mask == 0  # True 表示需要掩码

# 应用掩码
mask = causal_mask(seq_len)
output = scaled_dot_product_attention(Q, K, V, mask)
```

## 常见问题

### Q: 为什么 Attention 叫"注意力"？

因为它模拟了人类的选择性注意力——当我们阅读一段文字时，会根据当前理解的内容重点关注相关的词。Attention 机制让模型能够为不同位置分配不同的权重。

### Q: 如何确定 Attention 头数？

通常通过实验确定。更多的头数通常能捕捉更丰富的关系，但计算成本也更高。现代模型通常使用 16-96 个头。

### Q: Attention 权重可以解释吗？

可以，但需要谨慎：
- 可视化注意力权重可以了解模型关注什么
- 但这不代表模型"理解"了这些关系
- 注意力权重可能存在噪声