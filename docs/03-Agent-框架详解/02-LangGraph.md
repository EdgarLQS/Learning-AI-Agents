# LangChain 与 LangGraph

> LangChain 生态与 LangGraph 流程编排

## 一、LangChain 生态系统

### 1.1 发展历史

LangChain 由 Harrison Chase 于 2022 年 11 月创建，迅速成为最流行的 LLM 应用开发框架。

**发展阶段：**

| 阶段 | 时间 | 特点 |
|------|------|------|
| LangChain v1 | 2022-2023 | 链式（Chain）为核心 |
| LangChain v2 | 2023 年中 | Agent 系统成熟 |
| LangChain + LangGraph | 2024 年 | 图（Graph）工作流 |

### 1.2 LangChain 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                     LangChain 核心组件                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│   │   Models     │  │   Prompts    │  │   Chains     │      │
│   │  模型抽象层  │  │  提示词管理  │  │  链式调用    │      │
│   └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│   │    Agents    │  │    Memory    │  │   Tools      │      │
│   │  自主决策    │  │  记忆管理    │  │  工具集成    │      │
│   └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**核心组件说明：**

| 组件 | 功能 | 示例 |
|------|------|------|
| **Models** | 统一 LLM 接口 | ChatOpenAI, ChatAnthropic |
| **Prompts** | 提示词模板管理 | PromptTemplate, FewShotPrompt |
| **Chains** | 多步骤调用链 | LLMChain, SequentialChain |
| **Agents** | 自主决策执行 | ZeroShotAgent, ReAct Agent |
| **Memory** | 上下文持久化 | ConversationBufferMemory |
| **Tools** | 外部系统集成 | SearchTool, PythonREPLTool |

### 1.3 从 Chain 到 Graph 的演进

**LangChain Chain 的局限性：**

```
Chain 流程：线性执行
┌─────┐   ┌─────┐   ┌─────┐
│ A   │ → │ B   │ → │ C   │
└─────┘   └─────┘   └─────┘
```

- ❌ 难以表达复杂条件分支
- ❌ 不支持循环/迭代
- ❌ 状态管理不灵活
- ❌ 无法动态调整流程

**LangGraph 解决方案：**

```
Graph 流程：灵活的图结构
┌─────┐   ┌─────┐
│ A   │ → │ B   │
└─────┘   └─┬───┘
    ↑       │
    │   ┌───┴───┐
    └───│ 条件判断│
        └───┬───┘
            ↓
          ┌─────┐
          │ C   │
          └─────┘
```

- ✅ 支持任意图结构
- ✅ 原生支持循环
- ✅ 灵活的状态管理
- ✅ 支持条件分支

---

## 二、LangGraph 简介

### 2.1 什么是 LangGraph

LangGraph 是 LangChain 团队于 2024 年推出的 **Agent 流程编排框架**，基于**状态机**和**图论**设计，专门用于构建**有状态的多步骤工作流**。

### 2.2 为什么需要 LangGraph

| 场景 | LangChain | LangGraph |
|------|-----------|-----------|
| 简单问答 | ✅ 优秀 | ⚠️ 过度设计 |
| 多步骤对话 | ⚠️ 有限支持 | ✅ 原生支持 |
| 条件分支 | ❌ 困难 | ✅ 简单 |
| 循环/重试 | ❌ 不支持 | ✅ 原生支持 |
| 状态持久化 | ⚠️ 基础 | ✅ Checkpoints |
| 人类介入 | ❌ 困难 | ✅ 内置支持 |

### 2.3 核心设计理念

```
┌─────────────────────────────────────────────────────────────┐
│                    LangGraph 设计哲学                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   1. State-Centered（状态为中心）                            │
│      所有操作围绕状态展开                                    │
│                                                              │
│   2. Graph-Based（基于图）                                   │
│      用图结构表达复杂流程                                    │
│                                                              │
│   3. Cyclic Support（支持循环）                              │
│      原生支持循环和迭代                                      │
│                                                              │
│   4. Persistence-Ready（可持久化）                           │
│      内置状态保存与恢复                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、核心概念

### 3.1 State（状态）

State 是在节点之间传递的共享数据，使用 TypedDict 定义。

```python
from typing import TypedDict, List
from dataclasses import dataclass

# 方式 1: TypedDict
class AgentState(TypedDict):
    messages: List[str]
    context: dict
    step_count: int

# 方式 2: dataclass（推荐，支持默认值）
@dataclass
class AgentState:
    messages: List[str] = None
    context: dict = None
    step_count: int = 0
    errors: List[str] = None
```

**状态更新机制：**

```python
def node_a(state: AgentState) -> AgentState:
    # 返回的字典会合并到当前状态
    return {
        "messages": state["messages"] + ["Step A completed"],
        "step_count": state["step_count"] + 1
    }
```

### 3.2 Node（节点）

Node 是图中的基本执行单元，是一个接收状态并返回状态更新的函数。

```python
def research_node(state: AgentState) -> AgentState:
    """研究节点：执行搜索和信息收集"""
    query = state["messages"][-1]
    results = search_engine.search(query)
    return {
        "context": {"research_results": results},
        "messages": state["messages"] + [f"Found {len(results)} results"]
    }

def analysis_node(state: AgentState) -> AgentState:
    """分析节点：处理研究结果"""
    research = state["context"]["research_results"]
    analysis = llm.analyze(research)
    return {
        "context": {**state["context"], "analysis": analysis},
        "messages": state["messages"] + ["Analysis complete"]
    }
```

### 3.3 Edge（边）

Edge 决定节点之间的执行流程，分为三种类型：

#### 普通边（Normal Edge）

```python
graph.add_edge("node_a", "node_b")  # A 执行完后执行 B
```

#### 条件边（Conditional Edge）

```python
def route(state: AgentState) -> str:
    if state["step_count"] < 3:
        return "retry_node"
    else:
        return "final_node"

graph.add_conditional_edges(
    "decision_node",
    route,
    {
        "retry_node": "retry_node",
        "final_node": "final_node"
    }
)
```

#### 入口边（Entry Edge）

```python
graph.set_entry_point("start_node")  # 设置图的入口
```

### 3.4 Checkpoint（检查点）

Checkpoint 机制支持状态持久化，实现断点续传和时间旅行。

```python
from langgraph.checkpoint.memory import MemorySaver

# 配置 Checkpointer
memory = MemorySaver()
graph = StateGraph(AgentState)
# ... 添加节点和边
app = graph.compile(checkpointer=memory)

# 使用线程 ID 管理会话
config = {"configurable": {"thread_id": "user-123"}}

# 第一次运行
result1 = app.invoke({"messages": ["Hello"]}, config)

# 从检查点恢复（即使程序重启）
result2 = app.invoke({"messages": ["Continue"]}, config)
```

**Checkpoint 工作原理：**

```
┌─────────────────────────────────────────────────────────────┐
│                    Checkpoint 机制                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Step 1 ──→ State1 ──→ Checkpoint1                         │
│   Step 2 ──→ State2 ──→ Checkpoint2                         │
│   Step 3 ──→ State3 ──→ Checkpoint3                         │
│                                                              │
│   恢复：加载 Checkpoint2 → 从 State2 继续执行                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 四、工作流程与控制流

### 4.1 顺序执行

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  Start  │ → │  Node A │ → │  Node B │ → │  End    │
└─────────┘   └─────────┘   └─────────┘   └─────────┘
```

```python
from langgraph.graph import StateGraph, END

class SimpleState(TypedDict):
    result: str

def step_a(state):
    return {"result": "A done"}

def step_b(state):
    return {"result": state["result"] + " → B done"}

graph = StateGraph(SimpleState)
graph.add_node("step_a", step_a)
graph.add_node("step_b", step_b)
graph.set_entry_point("step_a")
graph.add_edge("step_a", "step_b")
graph.add_edge("step_b", END)

app = graph.compile()
```

### 4.2 条件分支

```
                    ┌─────────┐
              ┌────→│  Node B │
              │     └─────────┘
              │
┌─────────┐   │     ┌─────────┐
│  Start  │ → │     │  Node C │
└─────────┘   │     └─────────┘
              │
              │     ┌─────────┐
              └────→│  Node D │
                    └─────────┘
```

```python
def classify_intent(state):
    last_msg = state["messages"][-1]
    if "buy" in last_msg.lower():
        return "sales"
    elif "help" in last_msg.lower():
        return "support"
    else:
        return "general"

graph.add_node("classify", classify_intent)
graph.add_node("sales", sales_handler)
graph.add_node("support", support_handler)
graph.add_node("general", general_handler)

graph.set_entry_point("classify")
graph.add_conditional_edges(
    "classify",
    classify_intent,
    {
        "sales": "sales",
        "support": "support",
        "general": "general"
    }
)
graph.add_edge("sales", END)
graph.add_edge("support", END)
graph.add_edge("general", END)
```

### 4.3 循环/迭代

```
┌─────────┐   ┌─────────┐
│  Start  │ → │  Node A │
└─────────┘   └────┬────┘
       ↑           │
       │     ┌─────▼─────┐
       │     │  条件判断  │
       │     └─────┬─────┘
       │           │
       │    条件满足 │
       └───────────┘
```

```python
def check_continue(state):
    """检查是否继续循环"""
    if state["step_count"] < 5:
        return "continue"
    return "end"

graph.add_node("process", process_node)
graph.add_node("check", check_continue)

graph.set_entry_point("process")
graph.add_edge("process", "check")
graph.add_conditional_edges(
    "check",
    check_continue,
    {
        "continue": "process",  # 循环回 process
        "end": END
    }
)
```

### 4.4 并行处理

```python
from concurrent.futures import ThreadPoolExecutor

def parallel_process(state):
    """并行处理多个任务"""
    tasks = state["tasks"]
    results = {}

    with ThreadPoolExecutor() as executor:
        futures = {
            executor.submit(process_task, task): name
            for name, task in tasks.items()
        }
        for future in futures:
            name = futures[future]
            results[name] = future.result()

    return {"results": results}
```

---

## 五、代码示例

### 5.1 基础示例：简单问答流

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

class ChatState(TypedDict):
    messages: List[str]
    response: str

def receive_input(state: ChatState):
    user_input = state["messages"][-1]
    return {"response": f"Received: {user_input}"}

def send_response(state: ChatState):
    print(f"Bot: {state['response']}")
    return {}

# 构建图
graph = StateGraph(ChatState)
graph.add_node("receive", receive_input)
graph.add_node("send", send_response)
graph.set_entry_point("receive")
graph.add_edge("receive", "send")
graph.add_edge("send", END)

app = graph.compile()

# 运行
result = app.invoke({"messages": ["Hello!"]})
```

### 5.2 进阶示例：条件分支（客服路由）

```python
from typing import TypedDict, List, Literal

class SupportState(TypedDict):
    messages: List[str]
    category: str
    assigned_to: str

def classify_ticket(state: SupportState):
    content = state["messages"][-1].lower()
    if "bug" in content or "error" in content:
        return {"category": "technical"}
    elif "refund" in content or "payment" in content:
        return {"category": "billing"}
    else:
        return {"category": "general"}

def route_to_agent(state: SupportState):
    category = state["category"]
    if category == "technical":
        return "technical_team"
    elif category == "billing":
        return "billing_team"
    else:
        return "general_support"

def technical_team(state):
    return {"assigned_to": "Tech Team", "messages": ["Assigned to Tech Team"]}

def billing_team(state):
    return {"assigned_to": "Billing Team", "messages": ["Assigned to Billing Team"]}

def general_support(state):
    return {"assigned_to": "General Support", "messages": ["Assigned to General Support"]}

# 构建路由图
graph = StateGraph(SupportState)
graph.add_node("classify", classify_ticket)
graph.add_node("technical_team", technical_team)
graph.add_node("billing_team", billing_team)
graph.add_node("general_support", general_support)

graph.set_entry_point("classify")
graph.add_conditional_edges(
    "classify",
    route_to_agent,
    {
        "technical_team": "technical_team",
        "billing_team": "billing_team",
        "general_support": "general_support"
    }
)
graph.add_edge("technical_team", END)
graph.add_edge("billing_team", END)
graph.add_edge("general_support", END)

app = graph.compile()

# 运行
result = app.invoke({"messages": ["I found a bug in the login page!"]})
# 输出：assigned_to = "Tech Team"
```

### 5.3 进阶示例：循环重试（带错误处理）

```python
from typing import TypedDict, List, Optional

class RetryState(TypedDict):
    task: str
    attempts: int
    max_attempts: int
    result: Optional[str]
    errors: List[str]

def execute_task(state: RetryState):
    """执行任务（可能失败）"""
    import random
    if random.random() < 0.7:  # 70% 失败率模拟
        return {
            "errors": state["errors"] + ["Task failed, will retry..."],
            "result": None
        }
    return {"result": "Task completed successfully!", "errors": state["errors"]}

def check_retry(state: RetryState) -> Literal["retry", "success", "failed"]:
    if state["result"] is not None:
        return "success"
    if state["attempts"] >= state["max_attempts"]:
        return "failed"
    return "retry"

def retry_task(state: RetryState):
    return {"attempts": state["attempts"] + 1}

def handle_success(state: RetryState):
    return {"errors": state["errors"] + [f"Success after {state['attempts']} attempts"]}

def handle_failure(state: RetryState):
    return {"errors": state["errors"] + [f"Failed after {state['max_attempts']} attempts"]}

# 构建重试图
graph = StateGraph(RetryState)
graph.add_node("execute", execute_task)
graph.add_node("retry", retry_task)
graph.add_node("success", handle_success)
graph.add_node("failed", handle_failure)

graph.set_entry_point("execute")
graph.add_conditional_edges(
    "execute",
    check_retry,
    {
        "retry": "retry",
        "success": "success",
        "failed": "failed"
    }
)
graph.add_edge("retry", "execute")
graph.add_edge("success", END)
graph.add_edge("failed", END)

app = graph.compile()

# 运行
result = app.invoke({
    "task": "Process data",
    "attempts": 0,
    "max_attempts": 5,
    "result": None,
    "errors": []
})
```

### 5.4 实战示例：研究助手（多步骤 Agent）

```python
from typing import TypedDict, List, Optional
from langgraph.graph import StateGraph, END

class ResearchState(TypedDict):
    topic: str
    search_results: List[str]
    analysis: str
    report: str
    step: str

def search_researcher(state: ResearchState):
    """步骤 1: 搜索信息"""
    topic = state["topic"]
    # 模拟搜索
    results = [
        f"Article about {topic} - Source A",
        f"Research paper on {topic} - Source B",
        f"Blog post about {topic} - Source C"
    ]
    print(f"🔍 Searching for '{topic}'... Found {len(results)} results")
    return {"search_results": results, "step": "search_done"}

def analyze_researcher(state: ResearchState):
    """步骤 2: 分析搜索结果"""
    results = state["search_results"]
    # 模拟分析
    analysis = f"Analysis: Found {len(results)} relevant sources about '{state['topic']}'. Key themes identified."
    print(f"📊 Analyzing research...")
    return {"analysis": analysis, "step": "analysis_done"}

def report_writer(state: ResearchState):
    """步骤 3: 撰写报告"""
    report = f"""
# Research Report: {state['topic']}

## Summary
{state['analysis']}

## Sources
{chr(10).join('- ' + r for r in state['search_results'])}

## Conclusion
This report was generated by LangGraph Research Assistant.
    """.strip()
    print(f"📝 Writing report...")
    return {"report": report, "step": "report_done"}

# 构建研究助手
graph = StateGraph(ResearchState)
graph.add_node("search", search_researcher)
graph.add_node("analyze", analyze_researcher)
graph.add_node("write", report_writer)

graph.set_entry_point("search")
graph.add_edge("search", "analyze")
graph.add_edge("analyze", "write")
graph.add_edge("write", END)

app = graph.compile()

# 运行
result = app.invoke({"topic": "AI Agent Architecture", "step": ""})
print("\n" + "="*50)
print(result["report"])
```

---

## 六、高级特性

### 6.1 人类介入（Human-in-the-loop）

LangGraph 支持在关键节点暂停，等待人类确认或输入。

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List, Optional

class ApprovalState(TypedDict):
    action: str
    approved: Optional[bool]
    feedback: Optional[str]

def propose_action(state: ApprovalState):
    print(f"Proposed action: {state['action']}")
    return state

def wait_for_approval(state: ApprovalState):
    # 实际应用中，这里会等待用户输入
    # 使用 checkpointer 暂停，等待人类决策
    pass

def execute_action(state: ApprovalState):
    if state["approved"]:
        print(f"Executing: {state['action']}")
        return {"feedback": "Action executed"}
    else:
        print(f"Rejected: {state['action']}")
        return {"feedback": "Action rejected"}

graph = StateGraph(ApprovalState)
graph.add_node("propose", propose_action)
graph.add_node("wait", wait_for_approval)
graph.add_node("execute", execute_action)

graph.set_entry_point("propose")
graph.add_edge("propose", "wait")
graph.add_edge("wait", "execute")
graph.add_edge("execute", END)

# 配置 checkpointer 实现暂停
from langgraph.checkpoint.memory import MemorySaver
memory = MemorySaver()
app = graph.compile(checkpointer=memory)

# 暂停后，可以通过 update_state 恢复
config = {"configurable": {"thread_id": "approval-123"}}
app.invoke({"action": "Delete production database"}, config)
# 等待人类批准...
app.update_state(config, {"approved": True})  # 继续执行
```

### 6.2 时间旅行调试（Time Travel Debugging）

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app = graph.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "debug-session"}}

# 运行并保存状态
for step in app.stream({"messages": ["Hello"]}, config):
    print(step)

# 获取所有检查点
snapshots = list(app.get_state_history(config))

# 恢复到之前的状态
target_snapshot = snapshots[1]  # 恢复到第 1 步
app.update_state(config, target_snapshot.values)
```

### 6.3 流式输出

```python
# 流式获取中间状态
for step in app.stream({"messages": ["Start"]}, config):
    for node_name, output in step.items():
        print(f"[{node_name}] Output: {output}")

# 流式输出示例：
# [search] Output: {'search_results': [...]}
# [analyze] Output: {'analysis': '...'}
# [write] Output: {'report': '...'}
```

### 6.4 多 Assistant 协作

```python
class MultiAgentState(TypedDict):
    messages: List[str]
    current_agent: str
    results: dict

def agent_a(state):
    """研究 Agent"""
    return {"results": {"research": "Done"}, "current_agent": "agent_b"}

def agent_b(state):
    """写作 Agent"""
    return {"results": {**state["results"], "writing": "Done"}, "current_agent": "agent_c"}

def agent_c(state):
    """审核 Agent"""
    return {"results": {**state["results"], "review": "Done"}}

def route_next(state):
    return state["current_agent"]

graph = StateGraph(MultiAgentState)
graph.add_node("agent_a", agent_a)
graph.add_node("agent_b", agent_b)
graph.add_node("agent_c", agent_c)

graph.set_entry_point("agent_a")
graph.add_conditional_edges(
    "agent_a",
    route_next,
    {"agent_b": "agent_b"}
)
graph.add_conditional_edges(
    "agent_b",
    route_next,
    {"agent_c": "agent_c"}
)
graph.add_edge("agent_c", END)

app = graph.compile()
```

---

## 七、实际应用场景

### 7.1 客服对话系统

```
┌─────────────┐
│  客户消息   │
└──────┬──────┘
       │
┌──────▼──────┐
│  意图分类   │
└──────┬──────┘
       │
┌──────┼──────┬──────────┐
│      │      │          │
│      ▼      ▼          ▼
│  售前咨询  技术支持   投诉建议
│      │      │          │
│      └──────┴──────────┘
│             │
┌──────▼──────┐
│  生成回复   │
└──────┬──────┘
       │
┌──────▼──────┐
│  人工审核？  │──→ 是 ──→ [人工处理]
└──────┬──────┘
       │
       否
       │
┌──────▼──────┐
│  发送回复   │
└─────────────┘
```

### 7.2 代码审查工作流

```python
class CodeReviewState(TypedDict):
    code: str
    pr_description: str
    issues: List[str]
    suggestions: List[str]
    approval_status: str

def static_analysis(state):
    """静态代码分析"""
    # 运行 linter, security scan 等
    return {"issues": ["Line too long", "Missing docstring"]}

def security_review(state):
    """安全审查"""
    # 检查 SQL 注入、XSS 等
    return {"issues": state["issues"] + ["Potential SQL injection"]}

def logic_review(state):
    """逻辑审查"""
    # 检查业务逻辑
    return {"suggestions": ["Consider using a cache here"]}

def final_approval(state):
    """最终审批"""
    if len(state["issues"]) == 0:
        return {"approval_status": "APPROVED"}
    elif len(state["issues"]) <= 2:
        return {"approval_status": "CONDITIONAL"}
    else:
        return {"approval_status": "REJECTED"}

graph = StateGraph(CodeReviewState)
graph.add_node("static", static_analysis)
graph.add_node("security", security_review)
graph.add_node("logic", logic_review)
graph.add_node("approval", final_approval)

graph.set_entry_point("static")
graph.add_edge("static", "security")
graph.add_edge("security", "logic")
graph.add_edge("logic", "approval")
graph.add_edge("approval", END)
```

### 7.3 数据分析管道

```
数据收集 → 数据清洗 → 特征工程 → 模型训练 → 结果评估 → 报告生成
   │          │          │          │          │          │
   └──────────┴──────────┴──────────┴──────────┴──────────┘
                           错误处理与重试
```

---

## 八、与其他框架对比

### 8.1 综合对比表

| 特性 | LangGraph | AutoGPT | CrewAI |
|------|-----------|---------|--------|
| **核心范式** | 状态图 (Graph) | 递归循环 | 角色协作 |
| **流程控制** | 显式定义 | 自主决策 | 预定义流程 |
| **状态管理** | 内置 Checkpoint | 向量数据库 | 任务共享 |
| **多 Agent** | 手动实现 | 单 Agent | 原生支持 |
| **循环支持** | 原生支持 | 核心机制 | 顺序/并行 |
| **人类介入** | 内置支持 | 有限 | 有限 |
| **调试能力** | 时间旅行 | 日志 | 日志 |
| **学习曲线** | 中等 | 陡峭 | 平缓 |
| **适用场景** | 复杂工作流 | 探索性任务 | 多角色协作 |

### 8.2 选择建议

**选择 LangGraph 当：**
- ✅ 需要精确控制工作流程
- ✅ 流程包含条件分支和循环
- ✅ 需要状态持久化和恢复
- ✅ 需要人类介入审批

**选择 AutoGPT 当：**
- ✅ 任务目标明确但路径不清晰
- ✅ 需要 AI 自主探索和决策
- ✅ 可以接受较高的试错成本

**选择 CrewAI 当：**
- ✅ 需要多个专业角色协作
- ✅ 任务可以清晰拆分
- ✅ 追求快速搭建多 Agent 系统

---

## 九、最佳实践

### 9.1 状态设计原则

```python
# ✅ 好的状态设计
class GoodState(TypedDict):
    messages: List[str]           # 对话历史
    context: dict                 # 共享上下文
    current_step: str            # 当前步骤
    retry_count: int             # 重试计数

# ❌ 避免的状态设计
class BadState(TypedDict):
    # 问题：状态过于复杂，难以追踪
    data1: dict
    data2: dict
    data3: dict
    temp1: str
    temp2: str
    flag1: bool
    flag2: bool
    # ... 20+ 字段
```

**原则：**
1. 最小化状态字段
2. 使用嵌套结构组织相关数据
3. 为状态变更添加日志

### 9.2 避免无限循环

```python
# ✅ 带保护的循环
def check_continue(state):
    if state["retry_count"] >= MAX_RETRIES:
        return "give_up"
    if state["success"]:
        return "done"
    return "retry"

# ❌ 危险的循环（无退出条件）
def bad_check(state):
    if not state["success"]:
        return "retry"  # 可能永远失败
    return "done"
```

### 9.3 错误处理策略

```python
def robust_node(state):
    try:
        result = risky_operation()
        return {"result": result, "error": None}
    except Exception as e:
        return {"result": None, "error": str(e)}

def error_handler(state):
    if state["error"]:
        # 记录错误并决定下一步
        log_error(state["error"])
        return {"retry_count": state["retry_count"] + 1}
    return {}
```

### 9.4 性能优化建议

| 优化项 | 说明 |
|--------|------|
| 减少状态复制 | 大对象使用引用而非复制 |
| 异步节点 | IO 密集型操作使用 async/await |
| 批量处理 | 多个小任务合并处理 |
| 缓存结果 | 重复计算使用缓存 |

---

## 十、参考链接

- [LangChain 官网](https://www.langchain.com/)
- [LangChain GitHub](https://github.com/langchain-ai/langchain)
- [LangGraph 官方文档](https://langchain-ai.github.io/langgraph/)
- [LangGraph GitHub](https://github.com/langchain-ai/langgraph)
- [LangGraph 教程](https://langchain-ai.github.io/langgraph/tutorials/)

---

## 下一步

- [01-AutoGPT 详解](./01-AutoGPT%20详解.md) - 了解自主 Agent 先驱
- [03-CrewAI](./03-CrewAI.md) - 了解多 Agent 协作框架
- [00-框架对比](./00-框架对比.md) - 全面对比各框架
