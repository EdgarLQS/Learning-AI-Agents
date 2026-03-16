# 04-多Agent协作

多 Agent 协作让多个 Agent 能够协同工作，处理更复杂的任务。本文档详细介绍多 Agent 的通信协议和协作模式。

## 多 Agent 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    多 Agent 协作架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Agent 群组                              │   │
│  │                                                           │   │
│  │  ┌─────────┐     ┌─────────┐     ┌─────────┐            │   │
│  │  │ Agent A │ ←──→│ Manager │←──→│ Agent B │            │   │
│  │  │ (执行)  │     │  (协调)  │     │ (审查)  │            │   │
│  │  └─────────┘     └─────────┘     └─────────┘            │   │
│  │       ↓                                    ↓              │   │
│  │  ┌─────────┐                       ┌─────────┐           │   │
│  │  │ Agent C │                       │ Agent D │           │   │
│  │  │ (搜索)  │                       │ (测试)  │           │   │
│  │  └─────────┘                       └─────────┘           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    通信层                                  │   │
│  │  • 消息队列  • 事件总线  • 共享存储                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 通信协议

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Any, Optional
from datetime import datetime

class MessageType(Enum):
    """消息类型"""
    REQUEST = "request"       # 请求
    RESPONSE = "response"     # 响应
    EVENT = "event"          # 事件
    BROADCAST = "broadcast"  # 广播

@dataclass
class AgentMessage:
    """Agent 消息"""
    id: str
    sender: str
    receiver: str
    message_type: MessageType
    content: Any
    metadata: dict = field(default_factory=dict)
    timestamp: datetime = field(default_factory=datetime.now)

class MessageBus:
    """消息总线"""

    def __init__(self):
        self.queues: dict[str, list] = {}
        self.handlers: dict[str, list] = {}

    def send(self, message: AgentMessage):
        """发送消息"""
        receiver = message.receiver

        if receiver not in self.queues:
            self.queues[receiver] = []

        self.queues[receiver].append(message)

    def receive(self, agent_id: str) -> Optional[AgentMessage]:
        """接收消息"""
        if agent_id in self.queues and self.queues[agent_id]:
            return self.queues[agent_id].pop(0)
        return None

    def subscribe(self, agent_id: str, handler):
        """订阅消息"""
        if agent_id not in self.handlers:
            self.handlers[agent_id] = []
        self.handlers[agent_id].append(handler)
```

## 协作模式

### 1. 主管-工人模式

```python
class SupervisorWorker:
    """主管-工人模式"""

    def __init__(self):
        self.supervisor = None
        self.workers = []

    def set_supervisor(self, agent):
        """设置主管"""
        self.supervisor = agent

    def add_worker(self, agent, capabilities: list):
        """添加工人"""
        self.workers.append({
            "agent": agent,
            "capabilities": capabilities,
            "busy": False
        })

    async def delegate_task(self, task: str) -> dict:
        """分配任务"""

        # 1. 主管分析任务
        subtasks = await self.supervisor.analyze_task(task)

        # 2. 分配给合适的工人
        results = []
        for subtask in subtasks:
            worker = self._select_worker(subtask)
            if worker:
                result = await worker.execute(subtask)
                results.append(result)

        # 3. 主管汇总结果
        final_result = await self.supervisor.combine_results(results)

        return final_result

    def _select_worker(self, task) -> Optional[dict]:
        """选择合适的工人"""
        for worker in self.workers:
            if not worker["busy"]:
                # 检查能力匹配
                if any(cap in task.capabilities for cap in worker["capabilities"]):
                    return worker
        return None
```

### 2. 辩论模式

```python
class DebateMode:
    """辩论模式"""

    def __init__(self, agents: list):
        self.agents = agents
        self.rounds = 3

    async def debate(self, topic: str) -> dict:
        """执行辩论"""

        # 初始陈述
        statements = []
        for agent in self.agents:
            statement = await agent.think(topic)
            statements.append(statement)

        # 多轮辩论
        for round in range(self.rounds):
            for i, agent in enumerate(self.agents):
                # 基于其他人的陈述思考
                other_statements = [s for j, s in enumerate(statements) if j != i]
                response = await agent.respond(topic, other_statements)
                statements[i] = response

        # 最终结论
        conclusion = await self._make_conclusion(statements)

        return {
            "topic": topic,
            "statements": statements,
            "conclusion": conclusion
        }

    async def _make_conclusion(self, statements: list) -> str:
        """得出结论"""
        # 汇总各方观点
        return "综合以上讨论，结论是..."
```

## Agent 团队管理

```python
class AgentTeam:
    """Agent 团队"""

    def __init__(self):
        self.agents: Dict[str, Agent] = {}
        self.message_bus = MessageBus()
        self.task_queue = asyncio.Queue()

    def register_agent(self, agent_id: str, agent: Agent):
        """注册 Agent"""
        self.agents[agent_id] = agent
        agent.id = agent_id
        agent.set_message_bus(self.message_bus)

    async def run_task(self, task: str) -> dict:
        """运行任务"""

        # 启动所有 Agent
        tasks = []
        for agent in self.agents.values():
            tasks.append(asyncio.create_task(agent.run()))

        # 等待完成
        results = await asyncio.gather(*tasks, return_exceptions=True)

        return {"results": results}

    async def broadcast(self, message: dict):
        """广播消息"""
        for agent in self.agents.values():
            await agent.receive(message)
```

## 下一步

- [01-整体架构设计](./01-整体架构设计.md) - 了解架构设计
- [03-Agent-Core实现](./03-Agent-Core实现.md) - 了解核心实现
- [05-工业级架构](./05-工业级架构.md) - 了解工业级架构