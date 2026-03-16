# 03-Tool-Prompt设计

Tool Prompt 决定了 Agent 能否正确理解和使用工具。本文档详细介绍如何设计清晰、有效的工具提示。

## Tool Prompt 核心要素

```
┌─────────────────────────────────────────────────────────────────┐
│                    Tool Prompt 结构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 工具名称                                                   │   │
│  │ • 清晰、易懂、能表达功能                                    │   │
│  │ • 使用 snake_case 或 camelCase                            │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │ 工具描述                                                   │   │
│  │ • 功能说明：做什么                                         │   │
│  │ • 使用场景：什么时候用                                      │   │
│  │ • 注意事项：使用限制                                        │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │ 参数定义                                                   │   │
│  │ • 参数名、类型、描述                                        │   │
│  │ • 是否必填、默认值                                          │   │
│  │ • 枚举值、范围限制                                          │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │ 返回格式                                                   │   │
│  │ • 返回内容描述                                              │   │
│  │ • 格式说明                                                  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 工具描述示例

### 基础格式

```json
{
  "name": "read_file",
  "description": "读取指定文件的内容",
  "parameters": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "文件路径"
      }
    },
    "required": ["path"]
  }
}
```

### 详细格式

```json
{
  "name": "search_code",
  "description": "在代码库中搜索指定的文本或正则表达式",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "搜索关键词或正则表达式"
      },
      "file_pattern": {
        "type": "string",
        "description": "文件过滤模式，如 *.py, src/**/*.js",
        "default": "*"
      },
      "case_sensitive": {
        "type": "boolean",
        "description": "是否区分大小写",
        "default": true
      },
      "max_results": {
        "type": "integer",
        "description": "最大返回结果数",
        "default": 50,
        "minimum": 1,
        "maximum": 100
      }
    },
    "required": ["query"]
  },
  "returns": {
    "type": "array",
    "description": "匹配结果列表，每项包含文件路径、行号和内容"
  }
}
```

## 工具描述最佳实践

### 1. 使用自然的语言

```python
# ❌ 不好：过于技术化
"执行 shell 命令，返回 subprocess.CompletedProcess 对象"

# ✅ 好：清晰易懂
"执行系统命令，如运行脚本、安装依赖等。返回命令的输出结果。"
```

### 2. 包含使用场景

```python
# ❌ 不好：只说明功能
"搜索代码"

# ✅ 好：说明使用场景
"在代码库中搜索文本。当你需要：查找函数定义位置、了解某个符号的使用、定位特定代码时使用。"
```

### 3. 明确参数含义

```python
# ❌ 不好：参数描述模糊
"n: int 结果数量"

# ✅ 好：参数描述清晰
"n: int 返回结果的数量，默认为 5，最大 20"
```

### 4. 提供示例

```python
TOOL_PROMPT = """search_code - 搜索代码

功能：在代码库中搜索文本

参数：
- query: 搜索关键词（必填）
- file_pattern: 文件过滤模式（可选，默认 *）

示例：
搜索 Python 文件中的函数：
  query: "def calculate"
  file_pattern: "*.py"

搜索特定目录：
  query: "class User"
  file_pattern: "src/models/*.py"
"""
```

## 复杂工具描述

### 带依赖的工具

```python
{
  "name": "read_file",
  "description": "读取文件内容",
  "parameters": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "文件路径"
      }
    },
    "required": ["path"]
  }
}

{
  "name": "edit_file",
  "description": "编辑文件内容",
  "parameters": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "文件路径（必须先使用 read_file 读取）"
      },
      "operation": {
        "type": "string",
        "enum": ["replace", "insert", "delete"],
        "description": "操作类型"
      },
      "old_string": {
        "type": "string",
        "description": "要替换的文本（用于 replace 和 delete）"
      },
      "new_string": {
        "type": "string",
        "description": "要插入或替换成的文本"
      }
    },
    "required": ["path", "operation"]
  },
  "notes": "必须先读取文件才能编辑"
}
```

### 多步骤工具

```python
{
  "name": "run_tests",
  "description": "运行测试套件",
  "parameters": {
    "type": "object",
    "properties": {
      "test_path": {
        "type": "string",
        "description": "测试文件或目录路径"
      },
      "framework": {
        "type": "string",
        "enum": ["pytest", "jest", "unittest", "auto"],
        "description": "测试框架，auto 表示自动检测"
      },
      "verbose": {
        "type": "boolean",
        "description": "是否显示详细输出",
        "default": true
      },
      "coverage": {
        "type": "boolean",
        "description": "是否生成覆盖率报告",
        "default": false
      }
    }
  },
  "returns": {
    "description": "测试结果摘要，包括通过/失败数量和失败的测试详情"
  },
  "examples": [
    {
      "input": {"test_path": "tests/", "framework": "pytest"},
      "description": "运行所有测试"
    },
    {
      "input": {"test_path": "tests/test_user.py", "coverage": true},
      "description": "运行单个文件并生成覆盖率"
    }
  ]
}
```

## 工具分组

### 按功能分组

```python
# 文件操作工具组
FILE_TOOLS = [
    {
        "name": "read_file",
        "description": "读取文件内容",
        "category": "file"
    },
    {
        "name": "write_file",
        "description": "创建或覆盖文件",
        "category": "file"
    },
    {
        "name": "list_directory",
        "description": "列出目录内容",
        "category": "file"
    }
]

# 代码搜索工具组
SEARCH_TOOLS = [
    {
        "name": "grep",
        "description": "搜索文本",
        "category": "search"
    },
    {
        "name": "find_definitions",
        "description": "查找符号定义",
        "category": "search"
    }
]
```

### 分组提示

```
可用工具（按功能分组）：

【文件操作】
- read_file: 读取文件
- write_file: 写入文件
- list_directory: 列出目录

【代码搜索】
- grep: 搜索文本
- find_definitions: 查找定义

【命令执行】
- bash: 执行 shell 命令
- run_tests: 运行测试
```

## 动态工具描述

### 基于上下文生成

```python
class DynamicToolDescriber:
    """动态工具描述"""

    def __init__(self, project_context: dict):
        self.context = project_context

    def get_tool_description(self, tool: Tool) -> str:
        """生成工具描述"""

        base_desc = tool.description

        # 添加上下文相关提示
        if tool.name == "read_file":
            # 添加项目特定的文件扩展名提示
            extensions = self.context.get("file_extensions", [])
            if extensions:
                base_desc += f"\n项目常见文件类型: {', '.join(extensions)}"

        if tool.name == "run_tests":
            # 添加测试命令提示
            test_cmd = self.context.get("test_command")
            if test_cmd:
                base_desc += f"\n项目默认测试命令: {test_cmd}"

        return base_desc
```

## 验证工具描述

```python
class ToolDescValidator:
    """工具描述验证器"""

    def validate(self, tool_schema: dict) -> list:
        """验证工具描述"""
        errors = []

        # 检查必填字段
        required = ["name", "description", "parameters"]
        for field in required:
            if field not in tool_schema:
                errors.append(f"缺少必填字段: {field}")

        # 检查参数定义
        if "parameters" in tool_schema:
            params = tool_schema["parameters"]
            if "properties" not in params:
                errors.append("参数缺少 properties 定义")

            # 检查必填参数
            required_params = params.get("required", [])
            for param in required_params:
                if param not in params.get("properties", {}):
                    errors.append(f"必填参数 {param} 未定义")

        return errors
```

## 下一步

- [01-Prompt 设计基础](./01-Prompt 设计基础.md) - 了解基础概念
- [02-ReAct与CoT模式](./02-ReAct与CoT模式.md) - 了解推理模式
- [04-System-Prompt最佳实践](./04-System-Prompt最佳实践.md) - 了解系统提示