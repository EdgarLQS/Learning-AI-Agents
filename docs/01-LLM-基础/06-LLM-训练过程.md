# LLM 训练过程

## 6.1 训练流程概述

LLM 的训练通常分为三个阶段：

```
预训练 (Pretraining) → 监督微调 (SFT) → 对齐微调 (RLHF/DPO)
```

### 训练数据规模

| 阶段 | 数据量 | 成本 |
|------|--------|------|
| 预训练 | 兆亿级 tokens | 数百万-数亿美元 |
| SFT | 百万级样本 | 数十万-数百万美元 |
| RLHF | 十万级偏好对 | 数万-数十万美元 |

## 6.2 预训练（Pretraining）

### 目标

学习语言的**通用表示**，通过预测下一个 token（Next Token Prediction）。

### 训练数据来源

```
1. 网页文本 (Common Crawl, WebText)
2. 书籍 (BooksCorpus, Pile)
3. 代码 (GitHub, Stack Exchange)
4. 维基百科
5. 新闻文章
6. 学术论文
7. 对话数据
```

### 训练数据量

| 模型 | 训练 token 数 |
|------|--------------|
| GPT-3 | 175B |
| GPT-3.5 | ~300B |
| GPT-4 | ~1-2T |
| LLaMA 3 8B | 15T |
| LLaMA 3 70B | 15T |

### 训练目标

```
输入序列: "今天天气很好，我们去"
目标: "去公园散步"

Loss = CrossEntropy(predicted_token_id, actual_token_id)

训练目标：最小化 Loss，使预测更准确
```

### 训练过程

```
1. 数据分批 (Batch)
   - 将大量文本分成小批次
   - 每个批次包含多个序列

2. Token 化
   - 将文本转换为 token ID

3. 前向传播
   - 计算预测概率分布

4. 计算 Loss
   - 比较预测与真实 next token

5. 反向传播
   - 计算梯度并更新参数

6. 重复以上步骤
   - 遍历整个数据集多个 epoch
```

### 训练配置

| 参数 | 典型值 |
|------|--------|
| Batch Size | 1M+ tokens |
| Learning Rate | 1e-4 ~ 3e-4 |
| 训练步数 | 100K ~ 1M+ |
| GPU 数量 | 数千到数万 |
| 训练时间 | 数周到数月 |

## 6.3 词表扩展与特殊 Token

### 词表扩展

预训练前需要确定词表：

```python
# 示例：LLaMA 词表扩展
tokenizer.add_tokens(["<|system|>", "<|user|>", "<|assistant|>"])
tokenizer.add_special_tokens({"pad_token": "<|pad|>"})
```

### 特殊 Token

| Token | 作用 |
|-------|------|
| `<\|pad\|>` | 序列填充 |
| `<\|bos\|>` | 序列开始 |
| `<\|eos\|>` | 序列结束 |
| `<\|unk\|>` | 未知词 |
| `<\|sep\|>` | 对话分隔 |

## 6.4 监督微调（SFT, Supervised Fine-Tuning）

### 目标

让模型学会**遵循指令**，从通用语言模型转变为助手。

### 数据格式

```
示例1:
Input: "请解释什么是量子计算"
Output: "量子计算是一种..."

示例2:
Input: "写一首关于春天的诗"
Output: "春风拂面..."
```

### 数据量

| 模型 | SFT 数据量 |
|------|-----------|
| InstructGPT | 13K |
| LLaMA | 若干K |
| Claude | 未公开 |

### 训练方式

```
与预训练相同，但：
- 数据是指令-响应对
- 只计算 response 部分的 loss
- 使用更小的学习率
```

## 6.5 RLHF（基于人类反馈的强化学习）

### 为什么需要 RLHF

SFT 存在的问题：
- 可能生成有害内容
- 不够helpful
- 回答风格不符合人类偏好

RLHF 通过人类反馈来改进模型。

### RLHF 流程

```
1. 收集人类偏好数据
   - 给模型输出打分
   - 排序哪个更好

2. 训练奖励模型 (Reward Model)
   - 输入: (prompt, response)
   - 输出: 质量分数

3. 使用 PPO 优化
   - 目标: 最大化奖励
   - 约束: 不偏离 SFT 模型太远
```

### 奖励模型训练

```python
# 简化的奖励模型训练
def train_reward_model(prompts, responses_a, responses_b, preferences):
    """
    preferences: 哪个响应更好 (0 或 1)
    """
    score_a = reward_model(prompts, responses_a)
    score_b = reward_model(prompts, responses_b)

    # 使用对比损失
    loss = -log(sigmoid(score_b - score_a)) if preferences else ...
```

### PPO (Proximal Policy Optimization)

```python
# PPO 训练目标
def ppo_loss(policy_logits, old_logits, advantages, response_mask):
    """
    policy_logits: 当前模型输出
    old_logits: SFT 模型输出
    advantages: 奖励模型给的分数
    """
    # 1. 策略梯度损失
    ratio = exp(policy_logits - old_logits)
    policy_loss = -min(ratio * advantages, clip(ratio) * advantages)

    # 2. KL 散度惩罚（防止偏离 SFT 太远）
    kl_loss = KL(policy || sft_model)

    return policy_loss + kl_coef * kl_loss
```

## 6.6 DPO（Direct Preference Optimization）

### RLHF 的简化版本

不需要单独的奖励模型，直接使用偏好数据进行优化：

```python
# DPO 损失函数
def dpo_loss(policy_logits, chosen_logits, rejected_logits):
    """
    chosen: 人类选择的响应
    rejected: 人类拒绝的响应
    """
    # 最大化chosen与rejected的差异
    loss = -log(sigmoid(chosen_logits - rejected_logits))
```

### DPO vs RLHF

| 方面 | RLHF | DPO |
|------|------|-----|
| 训练复杂度 | 高（需要PPO） | 低（简单SFT） |
| 奖励模型 | 需要 | 不需要 |
| 效果 | 好 | 相当或更好 |
| 稳定性 | 难调优 | 更稳定 |

## 6.7 训练稳定性

### 常见问题

#### 梯度消失/爆炸

```
解决方案：
- 残差连接
- Layer Normalization
- 梯度裁剪 (max_norm=1.0)
```

#### Loss 尖峰

```
解决方案：
- 学习率 warm-up
- 梯度累积
- 混合精度训练
```

#### 训练不稳定

```
解决方案：
- 调整学习率
- 检查数据质量
- 使用稳定版本的网络结构
```

### 学习率调度

```python
# 典型的学习率调度
learning_rate_schedule = {
    "warmup_steps": 2000,      # 预热步数
    "peak_lr": 3e-4,          # 峰值学习率
    "min_lr": 1e-5,           # 最小学习率
    "total_steps": 100000     # 总步数
}

# Cosine 衰减
lr = min_lr + (peak_lr - min_lr) * 0.5 * (1 + cos(π * step / total_steps))
```

## 6.8 训练技巧

### 混合精度训练

```
使用 FP16/BF16 加速：
- 减少显存使用
- 加速计算
- 保持精度
```

### 梯度累积

```
当显存不足时：
- 累计多个小批次的梯度
- 再更新参数
- 等效于大批次训练
```

### 分布式训练

```
数据并行：
- 多个 GPU 处理不同数据
- 同步梯度

模型并行：
- 模型分布在多个 GPU
- 适合超大模型
```

## 常见问题

### Q: 预训练需要多少数据？

越多越好。现代模型通常使用数万亿到数十万亿 tokens。

### Q: SFT 和 RLHF 哪个更重要？

两者都很重要：
- SFT 让模型学会回答问题
- RLHF 让模型回答得更好、更安全

### Q: 为什么 RLHF 效果这么好？

因为它直接优化人类偏好，而不是简单的语言模型目标。