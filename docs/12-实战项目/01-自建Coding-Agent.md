# 01-自建Coding Agent

> 从零构建一个实用的 Coding Agent

## 项目概述

本项目旨在构建一个类似 Claude Code 的编程助手，能够：
- 理解代码仓库结构
- 读取和修改代码文件
- 执行命令和运行测试
- 自主完成编程任务

## 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      Coding Agent 架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                      User Interface                        │ │
│  │                   (CLI / Web UI / API)                    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                       Agent Core                           │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │ │
│  │  │   Planner   │  │   Executor  │  │    Memory          │  │ │
│  │  │  (任务规划)  │  │  (执行循环)  │  │   (上下文管理)      │  │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                      Tool System                          │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐  │ │
│  │  │ File   │ │ Search │ │  Git   │ │ Terminal│ │ Linter   │  │ │
│  │  │ System │ │        │ │        │ │         │ │ /Tests   │  │ │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └──────────┘  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                            ↓                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    Code Index System                       │ │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐     │ │
│  │  │  Code      │  │   AST      │  │    Embedding      │     │ │
│  │  │  Parser    │  │  Analyzer  │  │    (Vector Store) │     │ │
│  │  └────────────┘  └────────────┘  └────────────────────┘     │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 技术选型

### LLM 选择

| 模型 | 特点 | 适用场景 |
|------|------|----------|
| Claude 3.5 Sonnet | 代码能力强，性价比高 | 主要使用 |
| GPT-4o | 全面的能力 | 备选/对比 |
| Claude 3 Haiku | 速度快，价格低 | 简单任务 |

### 代码索引

| 方案 | 优点 | 缺点 |
|------|------|------|
| codebook (Anthropic) | 专为代码设计 | 需 API |
| CodeBERT | 开源本地 | 效果一般 |
| text-embedding-3-small | 通用，简单 | 非代码专用 |

### 存储

| 组件 | 方案 |
|------|------|
| 向量存储 | SQLite + json (简单) / Chroma (生产) |
| 文件索引 | 基于 Walk + glob |
| 配置存储 | YAML / JSON 文件 |

## 核心模块实现

### 1. 工具系统

```python
# tools/base.py
from abc import ABC, abstractmethod
from typing import Any, Dict

class BaseTool(ABC):
    name: str
    description: str
    parameters: Dict

    @abstractmethod
    def execute(self, **kwargs) -> Any:
        pass

# tools/file_tool.py
class FileTool(BaseTool):
    name = "read_file"
    description = "读取文件内容"

    def execute(self, path: str) -> str:
        with open(path, 'r', encoding='utf-8') as f:
            return f.read()
```

### 2. 代码理解

```python
# code_indexer/indexer.py
import os
import ast
from pathlib import Path

class CodeIndexer:
    def __init__(self, root_path: str):
        self.root = root_path
        self.files = {}
        self.ast_cache = {}

    def index(self):
        """索引整个代码库"""
        for root, dirs, files in os.walk(self.root):
            # 跳过不需要的目录
            dirs[:] = [d for d in dirs if not d.startswith('.')
                      and d not in ['node_modules', '__pycache__', 'venv']]

            for file in files:
                if self._should_index(file):
                    path = os.path.join(root, file)
                    self.files[path] = self._read_file(path)

    def _should_index(self, file: str) -> bool:
        """判断是否应该索引"""
        return file.endswith(('.py', '.js', '.ts', '.go', '.rs', '.java'))

    def get_relevant_files(self, query: str) -> list:
        """获取相关文件"""
        # 简单实现：基于关键词匹配
        # 生产环境应使用 Embedding 语义搜索
        return list(self.files.keys())[:10]
```

### 3. Agent 核心循环

```python
# agent/core.py
class CodingAgent:
    def __init__(self, llm, tools, code_indexer):
        self.llm = llm
        self.tools = tools
        self.indexer = code_indexer
        self.messages = []
        self.max_iterations = 100

    async def run(self, task: str) -> dict:
        """运行 Agent"""
        self.messages = [{"role": "user", "content": task}]

        for i in range(self.max_iterations):
            # 1. LLM 决定下一步
            response = await self.llm.chat(self.messages)

            # 2. 解析动作
            action = self._parse_action(response)

            # 3. 执行动作
            result = await self._execute_action(action)

            # 4. 添加到上下文
            self.messages.append({
                "role": "assistant",
                "content": f"Action: {action}\nResult: {result}"
            })

            # 5. 检查是否完成
            if self._is_complete(response):
                return {"status": "success", "result": result}

        return {"status": "max_iterations"}

    def _parse_action(self, response: dict) -> dict:
        """解析 LLM 响应为动作"""
        # 提取 thought, action, action_input
        pass

    async def _execute_action(self, action: dict) -> str:
        """执行动作"""
        tool_name = action.get("tool")
        tool = self.tools.get(tool_name)

        if tool:
            return tool.execute(**action.get("parameters", {}))
        return f"Unknown tool: {tool_name}"
```

### 4. Patch 生成

```python
# patch/generator.py
import difflib

class PatchGenerator:
    @staticmethod
    def generate(old_content: str, new_content: str, path: str) -> str:
        """生成 unified diff 格式的 patch"""
        diff = difflib.unified_diff(
            old_content.splitlines(keepends=True),
            new_content.splitlines(keepends=True),
            fromfile=f'a/{path}',
            tofile=f'b/{path}',
            lineterm=''
        )
        return ''.join(diff)

    @staticmethod
    def apply(patch: str, target_dir: str):
        """应用 patch"""
        import subprocess
        result = subprocess.run(
            ['patch', '-p1'],
            input=patch,
            capture_output=True,
            text=True,
            cwd=target_dir
        )
        return result.returncode == 0
```

## 实施步骤

### Phase 1: 最小可行版本

1. **基础工具**: 读取文件、写入文件、列出目录
2. **简单 Agent**: 接收任务 → 执行工具 → 返回结果
3. **测试**: 用简单任务验证 (如"读取 README.md")

### Phase 2: 代码理解

1. **文件索引**: 扫描项目结构
2. **代码解析**: 使用 AST 分析 Python 代码
3. **语义搜索**: 基于 Embedding 的相关文件检索

### Phase 3: 增强能力

1. **更多工具**: Git、Terminal、Linter、测试运行器
2. **错误处理**: 工具失败重试、超时处理
3. **自我反思**: 失败后重新规划

### Phase 4: 生产就绪

1. **Checkpoint**: 保存/恢复执行状态
2. **日志系统**: 详细记录执行过程
3. **配置管理**: 灵活的参数配置

## 评估与测试

### 测试用例

```python
# tests/test_basic.py
def test_read_file():
    agent = CodingAgent(llm, tools, indexer)
    result = agent.run("读取 main.py 文件")
    assert "def main" in result

def test_modify_file():
    agent = CodingAgent(llm, tools, indexer)
    result = agent.run("在 main.py 末尾添加 hello 函数")
    assert "def hello" in read_file("main.py")
```

### 评估指标

| 指标 | 说明 |
|------|------|
| 成功率 | 完成任务的比例 |
| 步数效率 | 平均消耗的步骤数 |
| 准确率 | 代码修改正确的比例 |
| 恢复率 | 错误后成功恢复的比例 |

## 进阶优化

### 1. 多模态理解
- 支持图片输入（UI 截图）
- 理解代码可视化

### 2. 多 Agent 协作
- 专门的代码审查 Agent
- 专门的测试 Agent

### 3. 学习与适应
- 从历史任务中学习
- 用户偏好学习

## 参考资源

- [Anthropic Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview)
- [OpenAI Codex](https://openai.com/blog/openai-codex)
- [Tree-sitter](https://tree-sitter.github.io/tree-sitter/) - 代码解析

## 下一步

- [02-案例库](./02-案例库.md) - 常见场景与解决方案