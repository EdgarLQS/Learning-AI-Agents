# CrewAI

> 多 Agent 协作框架，让多个 AI Agent 像团队一样协同工作

## 简介

CrewAI 是一个用于构建**多 Agent 协作系统**的框架。它的设计理念是将复杂任务分配给多个专业 Agent，每个 Agent 扮演特定角色，通过协作完成复杂目标。

### 核心特性

- **多角色 Agent**: 支持定义不同角色的 AI Agent
- **任务分派**: 将大任务拆分为子任务分配给不同 Agent
- **顺序/并行执行**: 支持任务按顺序或并行执行
- **记忆共享**: Agent 之间可以共享上下文

## 核心概念

```
┌─────────────────────────────────────────────────────────────────┐
│                      CrewAI 架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                        Crew                               │   │
│   │                    (组织单元)                             │   │
│   └─────────────────────────────────────────────────────────┘   │
│              ↑                    ↑                    ↑         │
│   ┌─────────┴───────┐  ┌─────────┴───────┐  ┌─────────┴────────┐│
│   │    Agent 1     │  │    Agent 2     │  │    Agent 3        ││
│   │  角色: Researcher│  │  角色: Writer  │  │  角色: Editor      ││
│   │  目标: 调研    │  │  目标: 撰写    │  │  目标: 审核       ││
│   └────────────────┘  └────────────────┘  └───────────────────┘│
│              │                    │                    │         │
│              ↓                    ↓                    ↓         │
│   ┌───────────────────────────────────────────────────────────┐   │
│   │                      Tasks                                │   │
│   │         (Agent 执行的具体任务)                             │   │
│   └───────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Agent

```python
from crewai import Agent

researcher = Agent(
    role="Research Analyst",
    goal="Find accurate information about the topic",
    backstory="Expert at researching and gathering information",
    tools=[search_tool, scrape_tool]
)
```

### Task

```python
from crewai import Task

research_task = Task(
    description="Research AI trends in 2024",
    agent=researcher,
    expected_output="Comprehensive report on AI trends"
)
```

### Crew

```python
from crewai import Crew

crew = Crew(
    agents=[researcher, writer, editor],
    tasks=[research_task, write_task, edit_task],
    process="sequential"  # 或 "parallel"
)
```

## 任务流程

### Sequential (顺序执行)

```
Task1 → Task2 → Task3
  ↓       ↓       ↓
Agent1  Agent2  Agent3
```

### Hierarchical (层级执行)

```
        Supervisor
           │
    ┌──────┼──────┐
    ↓      ↓      ↓
Worker1  Worker2  Worker3
```

## 实际应用场景

- **内容创作团队**: 研究员 → 写手 → 编辑
- **代码审查团队**: 开发者 → 审查者 → 测试工程师
- **数据分析团队**: 收集者 → 分析者 → 可视化工程师

## 与其他框架对比

| 特性 | CrewAI | LangGraph | AutoGPT |
|------|--------|-----------|---------|
| 多 Agent | ✅ 原生支持 | 需自行实现 | 有限 |
| 角色定义 | ✅ 完整 | 需自行定义 | 无 |
| 任务分派 | ✅ 自动 | 需手动 | 递归 |
| 学习门槛 | 低 | 中 | 中 |

## 参考链接

- [CrewAI 官网](https://www.crewai.com/)
- [CrewAI GitHub](https://github.com/crewAIInc/crewAI)
- [CrewAI 文档](https://docs.crewai.com/)

## 下一步

- [01-AutoGPT 详解](./01-AutoGPT%20详解.md) - 了解 AutoGPT
- [02-LangGraph](./02-LangGraph.md) - 了解 LangChain/LangGraph