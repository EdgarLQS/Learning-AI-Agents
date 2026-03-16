# 03-Agent-Core实现

Agent Core 是 Agent 系统的心脏，负责核心循环、任务调度和工具协调。本文档详细介绍 Agent Core 的设计和实现。

## Agent Core 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent Core 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    主循环 (Main Loop)                     │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │ 1. 思考 (Think) → 决定下一步行动                     │ │   │
│  │  │ 2. 行动 (Act) → 执行工具或返回结果                   │ │   │
│  │  │ 3. 观察 (Observe) → 评估行动结果                     │ │   │
│  │  │ 4. 决策 (Decide) → 决定是否继续                       │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    核心组件                               │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │思考引擎 │  │执行引擎 │  │状态管理 │  │工具调度 │    │   │
│  │  │(Planner)│  │(Executor)│  │(State)  │  │(Tools)  │    │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 主循环实现

```python
from typing import List, Dict, Any, Optional
from dataclasses import dataclass, field
from enum import Enum
import asyncio

class AgentAction(Enum):
    """Agent 可执行的动作"""
    THINK = "think"           # 思考（调用 LLM）
    USE_TOOL = "use_tool"    # 使用工具
    RESPOND = "respond"      # 返回结果
    FINISH = "finish"        # 完成

@dataclass
class Step:
    """执行步骤"""
    action: AgentAction
    thought: str = ""
    tool_name: Optional[str] = None
    tool_input: Optional[dict] = None
    observation: str = ""
    result: Any = None

class AgentCore:
    """Agent 核心"""

    def __init__(self, config: dict):
        self.config = config
        self.llm = None       # LLM 客户端
        self.tools = None    # 工具系统
        self.memory = None   # 记忆系统
        self.max_iterations = config.get("max_iterations", 100)

    async def run(self, task: str, context: dict = None) -> dict:
        """运行 Agent"""

        context = context or {}

        # 初始化
        steps_history = []
        messages = self._build_messages(task, context)

        # 主循环
        for iteration in range(self.max_iterations):
            # 1. 思考
            response = await self._think(messages)

            # 2. 解析动作
            action = self._parse_response(response)

            # 3. 执行动作
            if action.action == AgentAction.USE_TOOL:
                result = await self._use_tool(action)
                observation = f"工具 {action.tool_name} 返回: {result}"
            elif action.action == AgentAction.RESPOND:
                return {"status": "success", "result": action.thought}
            elif action.action == AgentAction.FINISH:
                return {"status": "completed", "result": action.thought}
            else:
                observation = action.thought

            # 4. 更新历史
            step = Step(
                action=action.action,
                thought=action.thought,
                tool_name=action.tool_name,
                tool_input=action.tool_input,
                observation=observation
            )
            steps_history.append(step)

            # 5. 更新消息
            messages.append({
                "role": "assistant",
                "content": action.thought
            })

            if observation:
                messages.append({
                    "role": "user",
                    "content": f"观察: {observation}"
                })

        # 达到最大迭代
        return {
            "status": "max_iterations",
            "steps": steps_history
        }

    def _build_messages(self, task: str, context: dict) -> List[dict]:
        """构建消息列表"""
        messages = [
            {
                "role": "system",
                "content": self._build_system_prompt(context)
            },
            {
                "role": "user",
                "content": task
            }
        ]
        return messages

    def _build_system_prompt(self, context: dict) -> str:
        """构建系统提示"""
        tool_descriptions = self.tools.get_descriptions()

        return f"""你是一个 AI Agent，可以帮助你完成各种任务。

可用工具:
{tool_descriptions}

使用规则:
1. 如果需要执行操作，使用工具
2. 如果已有足够信息，直接回答
3. 完成所有必要步骤后，明确结束

当前上下文: {context.get('cwd', '当前目录')}
"""

    async def _think(self, messages: list) -> dict:
        """思考 - 调用 LLM"""
        response = await self.llm.chat(messages)
        return response

    def _parse_response(self, response: dict) -> Step:
        """解析 LLM 响应"""
        content = response.get("content", "")

        # 简单解析：检查是否调用工具
        if "tool_call" in content.lower() or "使用工具" in content:
            # 提取工具名和参数
            # 简化处理
            return Step(
                action=AgentAction.USE_TOOL,
                thought=content,
                tool_name="bash",
                tool_input={"command": "echo parsed"}
            )
        elif "完成" in content or "结束" in content:
            return Step(action=AgentAction.FINISH, thought=content)
        else:
            return Step(action=AgentAction.RESPOND, thought=content)

    async def _use_tool(self, action: Step) -> Any:
        """使用工具"""
        return await self.tools.execute(action.tool_name, action.tool_input)
```

## 调度器实现

```python
class ToolScheduler:
    """工具调度器"""

    def __init__(self, tools, config: dict):
        self.tools = tools
        self.config = config
        self.execution_history = []

    async def schedule(self, task: str, available_tools: List[str]) -> dict:
        """调度工具执行"""

        # 1. 分析任务
        required_tools = self._analyze_requirements(task, available_tools)

        # 2. 排序（考虑依赖）
        execution_order = self._resolve_dependencies(required_tools)

        # 3. 执行
        results = []
        for tool_name in execution_order:
            result = await self._execute_with_retry(
                tool_name,
                self._extract_params(tool_name, task)
            )
            results.append({
                "tool": tool_name,
                "result": result
            })

            # 检查是否需要调整
            if self._should_adjust(task, results):
                # 重新规划
                execution_order = await self._replan(task, results)
                break

        return results

    def _analyze_requirements(self, task: str, tools: List[str]) -> List[str]:
        """分析需要的工具"""
        # 使用 LLM 或规则匹配
        required = []

        task_lower = task.lower()

        if "搜索" in task or "查找" in task:
            required.append("search")
        if "读取" in task or "查看" in task:
            required.append("read_file")
        if "修改" in task or "编辑" in task:
            required.extend(["read_file", "edit_file"])
        if "运行" in task or "执行" in task:
            required.append("bash")

        return list(set(required)) if required else tools[:1]

    def _resolve_dependencies(self, tools: List[str]) -> List[str]:
        """解析工具依赖"""
        # 简单策略：读取总是优先
        ordered = []
        remaining = list(tools)

        # 先放 read 类型的工具
        for tool in remaining[:]:
            if "read" in tool:
                ordered.append(tool)
                remaining.remove(tool)

        # 然后其他工具
        ordered.extend(remaining)

        return ordered

    async def _execute_with_retry(self, tool_name: str, params: dict) -> Any:
        """带重试的执行"""
        max_retries = self.config.get("tool_retries", 3)

        for attempt in range(max_retries):
            try:
                result = await self.tools.execute(tool_name, params)
                return result
            except Exception as e:
                if attempt == max_retries - 1:
                    raise
                await asyncio.sleep(2 ** attempt)

    def _extract_params(self, tool_name: str, task: str) -> dict:
        """提取工具参数"""
        # 简单实现
        return {"query": task}

    def _should_adjust(self, task: str, results: list) -> bool:
        """判断是否需要调整计划"""
        return False
```

## 状态管理

```python
from datetime import datetime
from typing import Any

class AgentState:
    """Agent 状态"""

    def __init__(self):
        self.status = "idle"
        self.current_task = None
        self.iteration = 0
        self.start_time = None
        self.end_time = None
        self.steps = []
        self.context = {}

    def start(self, task: str):
        """开始任务"""
        self.status = "running"
        self.current_task = task
        self.start_time = datetime.now()

    def add_step(self, step: Step):
        """添加步骤"""
        self.steps.append(step)
        self.iteration += 1

    def finish(self, result: Any):
        """完成任务"""
        self.status = "completed"
        self.end_time = datetime.now()

    def fail(self, error: str):
        """任务失败"""
        self.status = "failed"
        self.end_time = datetime.now()
        self.context["error"] = error

    def get_duration(self) -> float:
        """获取执行时长"""
        if self.start_time:
            end = self.end_time or datetime.now()
            return (end - self.start_time).total_seconds()
        return 0


class StateManager:
    """状态管理器"""

    def __init__(self):
        self.states = {}

    def create_state(self, session_id: str) -> AgentState:
        """创建状态"""
        state = AgentState()
        self.states[session_id] = state
        return state

    def get_state(self, session_id: str) -> AgentState:
        """获取状态"""
        return self.states.get(session_id)

    def update_state(self, session_id: str, state: AgentState):
        """更新状态"""
        self.states[session_id] = state
```

## 下一步

- [01-整体架构设计](./01-整体架构设计.md) - 了解架构设计
- [02-CLI实现](./02-CLI实现.md) - 了解 CLI 实现
- [04-多Agent协作](./04-多Agent协作.md) - 了解多 Agent 协作
- [05-工业级架构](./05-工业级架构.md) - 了解工业级架构