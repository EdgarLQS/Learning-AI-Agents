# 02-ReAct与CoT模式

ReAct 和 Chain of Thought (CoT) 是引导 LLM 进行复杂推理的两种核心模式。本文档详细介绍这两种模式的原理和实现。

## 推理模式概述

```
┌─────────────────────────────────────────────────────────────────┐
│                       推理模式对比                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────┐  ┌─────────────────────┐              │
│  │  Chain of Thought  │  │       ReAct         │              │
│  │      (CoT)          │  │                     │              │
│  ├─────────────────────┤  ├─────────────────────┤              │
│  │ 思考 → 思考 → 思考  │  │ 思考 → 行动 → 观察  │              │
│  │      ↓              │  │      ↓      ↓       │              │
│  │      答案           │  │      思考 → 行动... │              │
│  │                     │  │           ↓          │              │
│  │                     │  │           答案       │              │
│  └─────────────────────┘  └─────────────────────┘              │
│                                                                  │
│  适用：                      适用：                               │
│  • 数学推理                  • 需要外部工具                      │
│  • 逻辑分析                  • 需要实时信息                      │
│  • 逐步计算                  • 需要操作环境                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Chain of Thought (CoT)

### 原理

CoT 通过在 Prompt 中要求模型"逐步思考"，引导其生成中间推理步骤。

```python
# 不使用 CoT
prompt = "计算 123 * 456"

# 使用 CoT
prompt = """计算 123 * 456

请逐步计算：
1. 首先...
2. 然后...
3. 最后...

计算过程："""
```

### 效果

```
问题：农场有 15 只鸡，每只鸡每天下 1 个蛋。一个月（30天）能下多少个蛋？

不使用 CoT：
答案：450 个

使用 CoT：
1. 每天下蛋数 = 15 只 × 1 个 = 15 个
2. 一个月下蛋数 = 15 个 × 30 天 = 450 个
答案：450 个
```

### Zero-shot CoT

```python
# 只需在问题后加一句"逐步思考"
prompt = """问题：如果火车以每小时 80 公里的速度行驶 4 小时，它会行驶多远？
逐步思考："""

# 模型会自动生成推理步骤
```

### Few-shot CoT

```python
prompt = """请逐步解决以下问题。

示例问题：小明有 10 个苹果，给了小红 3 个，又买了 5 个，现在有多少个？
逐步思考：
1. 初始：10 个
2. 给出后：10 - 3 = 7 个
3. 买完后：7 + 5 = 12 个
答案：12 个

问题：商店进了 50 件商品，第一天卖了 15 件，第二天卖了 20 件，还剩多少件？
逐步思考："""
```

## ReAct 模式

### 原理

ReAct = Reasoning + Acting，让模型在推理过程中使用工具。

```python
REACT_PROMPT = """你是一个智能助手，可以通过思考和使用工具来完成任务。

对于每个任务，请按照以下格式输出：

Thought: [对当前情况的分析和下一步行动]
Action: [要使用的工具名称]
Action Input: [工具参数]
Observation: [工具返回的结果]
... (重复直到完成任务)
Answer: [最终答案]

开始任务！

{tool_descriptions}

用户任务：{task}
"""
```

### ReAct 实现

```python
class ReActAgent:
    """ReAct Agent"""

    def __init__(self, llm, tools):
        self.llm = llm
        self.tools = tools

    async def run(self, task: str, max_iterations: int = 10) -> dict:
        """运行 ReAct 循环"""

        # 构建提示
        prompt = self._build_prompt(task)

        history = []

        for i in range(max_iterations):
            # 调用 LLM
            response = await self.llm.chat(prompt)

            # 解析响应
            thought, action, action_input = self._parse_response(response)

            # 执行动作
            observation = await self._execute_action(action, action_input)

            # 更新历史
            history.append({
                "thought": thought,
                "action": action,
                "action_input": action_input,
                "observation": observation
            })

            # 更新提示
            prompt += self._format_step(history[-1])

            # 检查是否完成
            if self._is_completed(response):
                break

        return self._extract_answer(response)

    def _build_prompt(self, task: str) -> str:
        tool_desc = self._format_tools()
        return REACT_PROMPT.format(
            tool_descriptions=tool_desc,
            task=task
        )
```

## 组合模式：ReAct + CoT

### 原理

结合两种模式，让模型既能推理又能使用工具。

```python
REACT_COT_PROMPT = """你是一个智能助手，可以通过思考和使用工具来完成任务。

工作流程：
1. 逐步分析问题（Chain of Thought）
2. 决定需要使用的工具（ReAct）
3. 执行工具获取信息
4. 基于结果继续推理
5. 重复直到找到答案

请按以下格式输出：
Thought: [分析当前情况，思考下一步]
Action: [工具名称]
Action Input: [参数]
Observation: [工具返回结果]
...（重复）
Answer: [最终答案]

可用工具：
{tools}

开始解决问题：
{user_input}"""
```

### 使用示例

```
Thought: 用户想知道今天的天气。我需要先获取当前位置的天气信息。
Action: get_weather
Action Input: {"city": "北京"}
Observation: 今天北京天气晴朗，最高温度 25°C，最低温度 15°C。

Thought: 天气已获取，我可以回答用户的问题了。
Answer: 今天北京天气晴朗，温度 15-25°C，适合外出。
```

## 实践技巧

### 1. 清晰的结构标记

```python
# 使用清晰的标记帮助模型理解
prompt = """按照以下格式输出：

【思考】分析问题
【行动】决定工具
【执行】输入参数
【观察】查看结果
【结论】给出答案

---"""
```

### 2. 强制推理步骤

```python
prompt = """在给出最终答案之前，你必须：
1. 分析问题
2. 列出已知信息
3. 推导解决方案
4. 验证结果

问题：{question}"""
```

### 3. 工具可用性提示

```python
prompt = """你可以使用以下工具：
- search(query): 搜索信息
- calculator(expression): 计算数学表达式
- read_file(path): 读取文件

如果不需要使用工具，直接给出答案。
如果需要使用工具，按格式输出。

问题：{question}"""
```

## 常见问题

### 问题 1：模型跳过推理

```python
# 解决：在 Prompt 中强调推理
prompt = """重要：必须先进行推理才能给出答案。
不要直接给出答案，必须展示推理过程。

问题：{question}"""
```

### 问题 2：工具调用格式错误

```python
# 解决：提供详细格式说明和示例
prompt = """工具调用格式：
Action: 工具名
Action Input: {"参数名": "参数值"}

示例：
Action: search
Action Input: {"query": "Python 教程"}
"""
```

### 问题 3：无限循环

```python
# 解决：限制迭代次数
MAX_ITERATIONS = 10

for i in range(MAX_ITERATIONS):
    # 执行...
    if is_completed:
        break
```

## 下一步

- [01-Prompt 设计基础](./01-Prompt 设计基础.md) - 了解基础概念
- [03-Tool-Prompt设计](./03-Tool-Prompt设计.md) - 了解工具提示设计
- [04-System-Prompt最佳实践](./04-System-Prompt最佳实践.md) - 了解系统提示最佳实践