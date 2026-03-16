# LangChain 与 LangGraph

> LangChain 生态与 LangGraph 流程编排

## LangChain 简介

LangChain 是一个用于构建 AI 应用的开发框架，提供了：
- **LLM 接口**: 统一的各种 LLM 调用方式
- **提示词管理**: 模板化提示词构建
- **链式调用**: 将多个 LLM 调用串联
- **工具集成**: 与外部系统集成
- **内存管理**: 对话上下文持久化

## LangGraph 简介

LangGraph 是 LangChain 推出的 Agent 框架，专门用于构建**有状态的多步骤工作流**。

### 核心概念

- **Node**: 图中的节点，代表具体操作
- **Edge**: 节点之间的边，决定流程走向
- **State**: 在节点间传递的共享状态
- **Checkpoints**: 状态保存与恢复

### 工作原理

```
┌─────────────────────────────────────────────────────────────┐
│                      LangGraph 工作流程                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐ │
│   │  Start  │───→│  Node1  │───→│  Node2  │───→│  End    │ │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘ │
│                      ↑    │              │                  │
│                      │    ↓              │                  │
│                   ┌─────────┐    ┌─────────┐               │
│                   │  Node3  │───→│  Node4  │               │
│                   └─────────┘    └─────────┘               │
│                                                              │
│   State: {messages: [], context: {}, ...}                    │
│   Checkpoints: [state1, state2, state3, ...]                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 代码示例

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    next_action: str

def node1(state):
    return {"messages": [...], "next_action": "node2"}

def node2(state):
    return {"messages": [...], "next_action": "node3"}

# 构建图
graph = StateGraph(AgentState)
graph.add_node("node1", node1)
graph.add_node("node2", node2)
graph.set_entry_point("node1")
graph.add_edge("node1", "node2")
graph.add_edge("node2", END)

app = graph.compile()
```

## 与其他框架对比

| 特性 | LangChain | LangGraph |
|------|-----------|-----------|
| 工作流 | 链式 (Chain) | 图 (Graph) |
| 状态管理 | 基础 | 持久化 Checkpoints |
| 循环支持 | 有限 | 原生支持 |
| 适用场景 | 简单流程 | 复杂多步骤 Agent |

## 参考链接

- [LangChain 官网](https://www.langchain.com/)
- [LangChain GitHub](https://github.com/langchain-ai/langchain)
- [LangGraph 文档](https://langchain-ai.github.io/langgraph/)
- [LangGraph GitHub](https://github.com/langchain-ai/langgraph)

## 下一步

- [01-AutoGPT 详解](./01-AutoGPT%20详解.md) - 了解 AutoGPT
- [03-CrewAI](./03-CrewAI.md) - 了解 CrewAI