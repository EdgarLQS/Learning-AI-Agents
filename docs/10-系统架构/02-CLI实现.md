# 02-CLI实现

CLI（命令行界面）是 Agent 与用户交互的主要方式之一。本文档详细介绍 CLI 的设计和实现。

## CLI 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                       CLI 架构                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    用户输入                              │   │
│  │  $ agent "帮我重构用户认证模块"                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  输入解析器                               │   │
│  │  • 解析命令和参数                                         │   │
│  │  • 处理引号内的文本                                       │   │
│  │  • 识别标志选项                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  命令路由器                               │   │
│  │  • 匹配命令到处理器                                       │   │
│  │  • 参数验证                                              │   │
│  │  • 错误处理                                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   Agent Core                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  输出格式化器                             │   │
│  │  • 彩色输出                                              │   │
│  │  • 进度显示                                              │   │
│  │  • 错误提示                                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 命令行解析

```python
import shlex
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class Command:
    """命令"""
    name: str
    args: List[str]
    flags: dict
    raw_input: str

class CommandParser:
    """命令解析器"""

    def __init__(self):
        self.commands = {}
        self.aliases = {}

    def register_command(self, name: str, handler, aliases: List[str] = None):
        """注册命令"""
        self.commands[name] = handler
        if aliases:
            for alias in aliases:
                self.aliases[alias] = name

    def parse(self, raw_input: str) -> Command:
        """解析输入"""

        # 使用 shlex 正确处理引号
        try:
            tokens = shlex.split(raw_input)
        except ValueError:
            tokens = raw_input.split()

        if not tokens:
            return Command("", [], {}, raw_input)

        # 第一个 token 是命令
        cmd_name = tokens[0]

        # 检查别名
        cmd_name = self.aliases.get(cmd_name, cmd_name)

        # 解析参数和标志
        args = []
        flags = {}

        i = 1
        while i < len(tokens):
            token = tokens[i]

            if token.startswith("--"):
                # 长标志
                key = token[2:]
                if i + 1 < len(tokens) and not tokens[i + 1].startswith("-"):
                    flags[key] = tokens[i + 1]
                    i += 2
                else:
                    flags[key] = True
                i += 1

            elif token.startswith("-"):
                # 短标志
                key = token[1:]
                if i + 1 < len(tokens) and not tokens[i + 1].startswith("-"):
                    flags[key] = tokens[i + 1]
                    i += 2
                else:
                    flags[key] = True
                i += 1

            else:
                args.append(token)
                i += 1

        return Command(cmd_name, args, flags, raw_input)
```

## 交互模式

### 1. 交互式模式

```python
import sys

class InteractiveCLI:
    """交互式 CLI"""

    def __init__(self, agent_core):
        self.agent = agent_core
        self.parser = CommandParser()
        self.history = []
        self.running = False

    def start(self):
        """启动交互式 CLI"""
        self.running = True
        print("欢迎使用 AI Agent CLI")
        print("输入 'help' 查看帮助，输入 'exit' 退出")
        print()

        while self.running:
            try:
                user_input = input("\n> ")
                if not user_input.strip():
                    continue

                # 添加到历史
                self.history.append(user_input)

                # 处理命令
                await self.process_input(user_input)

            except KeyboardInterrupt:
                print("\n使用 'exit' 退出")
            except EOFError:
                break

        print("再见!")

    async def process_input(self, user_input: str):
        """处理用户输入"""

        # 检查特殊命令
        if user_input.strip() == "exit":
            self.running = False
            return

        if user_input.strip() == "help":
            self.print_help()
            return

        if user_input.strip() == "history":
            self.print_history()
            return

        # 执行 Agent
        result = await self.agent.run(user_input)

        # 显示结果
        self.print_result(result)

    def print_help(self):
        """打印帮助"""
        help_text = """
可用命令:
  exit           退出程序
  help           显示帮助
  history        显示命令历史

普通输入将直接传递给 Agent 处理。
        """
        print(help_text)

    def print_history(self):
        """打印历史"""
        for i, cmd in enumerate(self.history, 1):
            print(f"{i}: {cmd}")

    def print_result(self, result: dict):
        """打印结果"""
        if result.get("status") == "success":
            print(result.get("output", "完成"))
        else:
            print(f"错误: {result.get('error', '未知错误')}")
```

### 2. 单次执行模式

```python
class OneShotCLI:
    """单次执行 CLI"""

    def __init__(self, agent_core):
        self.agent = agent_core

    async def run(self, task: str, args: dict = None):
        """执行任务"""

        print(f"执行任务: {task}")
        print("-" * 50)

        # 显示进度
        with ProgressDisplay() as progress:
            result = await self.agent.run(task)

            progress.complete()

        # 显示结果
        self.display_result(result)

    def display_result(self, result: dict):
        """显示结果"""
        status = result.get("status")

        if status == "success":
            print("\n✅ 任务完成")
            if result.get("output"):
                print("\n输出:")
                print(result["output"])
        else:
            print("\n❌ 任务失败")
            if result.get("error"):
                print(f"\n错误: {result['error']}")
```

## 输出格式化

### 彩色输出

```python
class ColoredOutput:
    """彩色输出"""

    COLORS = {
        "red": "\033[91m",
        "green": "\033[92m",
        "yellow": "\033[93m",
        "blue": "\033[94m",
        "magenta": "\033[95m",
        "cyan": "\033[96m",
        "white": "\033[97m",
        "reset": "\033[0m"
    }

    @classmethod
    def colorize(cls, text: str, color: str) -> str:
        """添加颜色"""
        color_code = cls.COLORS.get(color, "")
        reset = cls.COLORS["reset"]
        return f"{color_code}{text}{reset}"

    @classmethod
    def success(cls, text: str):
        """成功消息"""
        return cls.colorize(f"✅ {text}", "green")

    @classmethod
    def error(cls, text: str):
        """错误消息"""
        return cls.colorize(f"❌ {text}", "red")

    @classmethod
    def warning(cls, text: str):
        """警告消息"""
        return cls.colorize(f"⚠️ {text}", "yellow")

    @classmethod
    def info(cls, text: str):
        """信息消息"""
        return cls.colorize(f"ℹ️ {text}", "cyan")
```

### 进度显示

```python
import time

class ProgressDisplay:
    """进度显示"""

    def __init__(self):
        self.start_time = None

    def __enter__(self):
        self.start_time = time.time()
        print("处理中...", end="\r")
        return self

    def __exit__(self, *args):
        pass

    def update(self, message: str):
        """更新进度"""
        print(f"处理中: {message}", end="\r")

    def complete(self):
        """完成"""
        elapsed = time.time() - self.start_time
        print(f"完成! 耗时: {elapsed:.2f}s")


class SpinnerProgress:
    """旋转器进度"""

    def __init__(self):
        self.frames = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"]
        self.current = 0

    def spin(self):
        """旋转"""
        frame = self.frames[self.current]
        self.current = (self.current + 1) % len(self.frames)
        print(f"{frame} 处理中...", end="\r")
```

## 完整 CLI 实现

```python
import argparse

class AgentCLI:
    """完整的 Agent CLI"""

    def __init__(self, agent_core):
        self.agent = agent_core
        self.parser = argparse.ArgumentParser(
            description="AI Agent CLI"
        )
        self._setup_parser()

    def _setup_parser(self):
        """设置解析器"""
        subparsers = self.parser.add_subparsers(dest="command")

        # run 命令
        run_parser = subparsers.add_parser("run", help="运行任务")
        run_parser.add_argument("task", help="任务描述")
        run_parser.add_argument("--interactive", "-i", action="store_true",
                               help="交互模式")

        # chat 命令
        chat_parser = subparsers.add_parser("chat", help="交互对话")

        # 通用参数
        for parser in [run_parser, chat_parser]:
            parser.add_argument("--verbose", "-v", action="store_true",
                              help="详细输出")
            parser.add_argument("--model", "-m", help="指定模型")

    async def run(self, args=None):
        """运行 CLI"""
        args = self.parser.parse_args(args)

        if args.command == "run":
            await self._run_task(args)
        elif args.command == "chat":
            await self._chat(args)
        else:
            self.parser.print_help()

    async def _run_task(self, args):
        """运行任务"""
        cli = OneShotCLI(self.agent)
        await cli.run(args.task)

    async def _chat(self, args):
        """交互对话"""
        cli = InteractiveCLI(self.agent)
        await cli.start()


# 入口点
def main():
    import asyncio

    # 初始化系统
    config = AgentConfig.from_file("config.json")
    system = AgentSystem(config)

    # 创建 CLI
    agent_core = system.container.resolve(AgentCore)
    cli = AgentCLI(agent_core)

    # 运行
    asyncio.run(cli.run())
```

## 下一步

- [01-整体架构设计](./01-整体架构设计.md) - 了解架构设计
- [03-Agent-Core实现](./03-Agent-Core实现.md) - 了解核心实现
- [04-多Agent协作](./04-多Agent协作.md) - 了解多 Agent 协作