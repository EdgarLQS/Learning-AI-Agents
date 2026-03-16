# AI Agent 工程师/架构师学习路径

本仓库记录 AI Agent 领域的系统化知识，帮助从零开始成为一名 AI Agent 工程师/架构师。

## 学习路线图

```
LLM 基础 → AI Agent 核心 → Agent 框架 → Prompt 工程 → 工具系统 → 记忆系统
                                              ↓
职业发展 ← 实战项目 ← 系统架构 ← Coding Agent ← 代码理解 ← 任务规划
```

## 目录结构

| 模块 | 内容 | 前置知识 | 状态 |
|------|------|----------|------|
| [01-LLM-基础](docs/01-LLM-基础/) | LLM 本质、Transformer、Embedding、Attention | 无 | ✅ |
| [02-AI-Agent-核心](docs/02-AI-Agent-核心/) | Agent 架构、核心能力、工作循环 | LLM 基础 | ✅ |
| [03-Agent-框架详解](docs/03-Agent-框架详解/) | AutoGPT、LangGraph、CrewAI | AI Agent 核心 | ✅ |
| [04-Prompt-工程](docs/04-Prompt-工程/) | System Prompt、ReAct、Tool Prompt | AI Agent 核心 | ✅ |
| [05-工具系统](docs/05-工具系统/) | 工具接口、调用机制、安全性、内置工具 | AI Agent 核心 | ✅ |
| [06-记忆系统](docs/06-记忆系统/) | 向量数据库、RAG、上下文管理、记忆架构 | LLM 基础 | ✅ |
| [07-代码理解系统](docs/07-代码理解系统/) | 代码切分、语义搜索、AST、Code Graph | 记忆系统 | ✅ |
| [08-Coding-Agent](docs/08-Coding-Agent/) | Claude Code 原理、Patch 生成、多文件重构 | 代码理解 | ✅ |
| [09-任务规划系统](docs/09-任务规划系统/) | 分层规划、任务拆解、状态机、长任务稳定性 | AI Agent 核心 | ✅ |
| [10-系统架构](docs/10-系统架构/) | 整体架构、CLI、Agent-Core、多Agent协作、工业级架构 | 以上全部 | ✅ |
| [11-瓶颈与挑战](docs/11-瓶颈与挑战/) | 规划不足、上下文爆炸、工具可靠性、成本速度、语言挑战 | 系统架构 | ✅ |
| [12-实战项目](docs/12-实战项目/) | 自建 Coding Agent、案例库 | 以上全部 | ✅ |
| [13-职业发展](docs/13-职业发展/) | 技能树、能力模型、学习资源 | 实战项目 | ✅ |

## 核心公式

```
AI Coding Agent = LLM + Prompt + Tools + Memory + Planning + Code Index
```

## 学习建议

1. **按顺序学习**：每个模块都有前置知识依赖
2. **理论与实践结合**：学习概念后在实战项目中实现
3. **持续更新**：AI Agent 领域发展迅速，持续补充新知识

## 知识来源

本仓库知识内容由学习者在学习 AI Agent 过程中持续积累和更新。
