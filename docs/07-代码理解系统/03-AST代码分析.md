# 03-AST代码分析

AST（抽象语法树）是代码的结构化表示，使 Agent 能够深入理解代码的语法和语义结构。本文档详细介绍 AST 分析技术及其应用。

## AST 概述

### 什么是 AST

AST 是源代码的树形表示，保留了代码的语法结构信息，去除了格式细节。

```
┌─────────────────────────────────────────────────────────────────┐
│                    代码 vs AST 对比                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  源代码:                         AST 结构:                      │
│  ┌──────────────────┐          ┌─────────────┐                  │
│  │ def add(a, b):   │          │ FunctionDef │                  │
│  │     return a + b │   ──→    │   add       │                  │
│  └──────────────────┘          │   ├─args    │                  │
│                                │   │   a      │                  │
│                                │   │   b      │                  │
│                                │   └─body    │                  │
│                                │     ├─Return│                  │
│                                │       └─BinOp│                  │
│                                │         ├─a │                  │
│                                │         └─b │                  │
│                                └─────────────┘                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 主要编程语言的 AST 工具

| 语言 | AST 工具 | 用途 |
|------|----------|------|
| Python | `ast` / `parser` | 标准库 |
| JavaScript | `@babel/parser`, `esprima` | 编译前端 |
| Java | `Javac`, `Eclipse JDT` | IDE |
| Go | `go/ast` | 标准库 |
| Rust | `syn`, `rustc-ap-rustc_ast` | 编译器 |
| C/C++ | `Clang`, `LLVM` | 编译器 |

## Python AST 分析

### 基础 AST 解析

```python
import ast
from typing import List, Dict, Any, Optional
from dataclasses import dataclass

@dataclass
class SymbolInfo:
    """符号信息"""
    name: str
    type: str  # function, class, variable, import
    line: int
    end_line: int
    file: str
    parent: Optional[str] = None

class PythonASTAnalyzer:
    """Python AST 分析器"""

    def __init__(self, file_path: str = ""):
        self.file_path = file_path
        self.tree = None
        self.symbols = []

    def parse(self, source: str):
        """解析源代码为 AST"""
        self.tree = ast.parse(source, filename=self.file_path)
        return self.tree

    def extract_symbols(self) -> List[SymbolInfo]:
        """提取所有符号（函数、类、变量）"""

        symbols = []

        for node in ast.walk(self.tree):
            if isinstance(node, ast.FunctionDef):
                symbols.append(SymbolInfo(
                    name=node.name,
                    type="function",
                    line=node.lineno,
                    end_line=node.end_lineno or node.lineno,
                    file=self.file_path
                ))

            elif isinstance(node, ast.AsyncFunctionDef):
                symbols.append(SymbolInfo(
                    name=node.name,
                    type="async_function",
                    line=node.lineno,
                    end_line=node.end_lineno or node.lineno,
                    file=self.file_path
                ))

            elif isinstance(node, ast.ClassDef):
                symbols.append(SymbolInfo(
                    name=node.name,
                    type="class",
                    line=node.lineno,
                    end_line=node.end_lineno or node.lineno,
                    file=self.file_path
                ))

            elif isinstance(node, (ast.Assign, ast.AnnAssign)):
                # 变量赋值
                for target in node.targets:
                    if isinstance(target, ast.Name):
                        symbols.append(SymbolInfo(
                            name=target.id,
                            type="variable",
                            line=node.lineno,
                            end_line=node.end_lineno or node.lineno,
                            file=self.file_path
                        ))

        self.symbols = symbols
        return symbols
```

### 依赖分析

```python
class DependencyAnalyzer:
    """依赖分析器"""

    def __init__(self):
        self.imports = []
        self.imports_from = []

    def analyze_imports(self, source: str) -> Dict[str, Any]:
        """分析导入语句"""

        tree = ast.parse(source)
        imports = []
        imports_from = []

        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for alias in node.names:
                    imports.append({
                        "module": alias.name,
                        "alias": alias.asname,
                        "line": node.lineno
                    })

            elif isinstance(node, ast.ImportFrom):
                imports_from.append({
                    "module": node.module,
                    "names": [alias.name for alias in node.names],
                    "level": node.level,  # 相对导入层级
                    "line": node.lineno
                })

        return {
            "imports": imports,
            "imports_from": imports_from,
            "all_imported_modules": self._get_all_modules(imports, imports_from)
        }

    def _get_all_modules(self, imports, imports_from) -> List[str]:
        """获取所有导入的模块"""
        modules = []

        for imp in imports:
            modules.append(imp["module"])

        for imp in imports_from:
            if imp["module"]:
                modules.append(imp["module"])

        return modules
```

### 控制流分析

```python
class ControlFlowAnalyzer:
    """控制流分析器"""

    def analyze(self, source: str) -> Dict[str, Any]:
        """分析控制流"""

        tree = ast.parse(source)

        # 收集所有函数
        functions = []
        for node in ast.walk(tree):
            if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
                functions.append(self._analyze_function(node))

        return {
            "functions": functions,
            "has_recursion": any(f["is_recursive"] for f in functions),
            "max_depth": max((f["max_depth"] for f in functions), default=0)
        }

    def _analyze_function(self, node: ast.FunctionDef) -> Dict[str, Any]:
        """分析单个函数"""

        # 计算圈复杂度（简化版）
        complexity = 1  # 基础复杂度

        for child in ast.walk(node):
            if isinstance(child, (ast.If, ast.While, ast.For)):
                complexity += 1
            elif isinstance(child, ast.BoolOp):
                complexity += len(child.values) - 1

        # 检测递归
        is_recursive = self._check_recursion(node)

        # 计算最大深度
        max_depth = self._calculate_depth(node)

        return {
            "name": node.name,
            "line": node.lineno,
            "complexity": complexity,
            "is_recursive": is_recursive,
            "max_depth": max_depth,
            "has_yield": isinstance(node, ast.AsyncFunctionDef) or
                        any(isinstance(n, (ast.Yield, ast.YieldFrom))
                            for n in ast.walk(node))
        }

    def _check_recursion(self, func_node: ast.FunctionDef) -> bool:
        """检测是否递归"""
        func_name = func_node.name

        for node in ast.walk(func_node):
            if isinstance(node, ast.Call):
                if isinstance(node.func, ast.Name) and node.func.id == func_name:
                    return True

        return False

    def _calculate_depth(self, node, current_depth=0):
        """计算嵌套深度"""
        max_depth = current_depth

        for child in ast.iter_child_nodes(node):
            depth = self._calculate_depth(child, current_depth + 1)
            max_depth = max(max_depth, depth)

        return max_depth
```

## TypeScript/JavaScript AST

### 使用 Babel 解析

```python
# 需要安装: pip install pybabel 或使用 Node.js babel

class JavaScriptASTAnalyzer:
    """JavaScript AST 分析器"""

    def __init__(self):
        pass

    def parse(self, source: str) -> dict:
        """解析 JS 代码（使用简化处理）"""
        # 实际使用可以用 pybabel 或调用 node 脚本
        return {"source": source, "type": "JavaScript"}

    def extract_functions(self, source: str) -> List[Dict]:
        """提取函数"""

        # 使用正则做简化分析
        import re

        # 匹配函数声明
        func_patterns = [
            r'function\s+(\w+)\s*\([^)]*\)',  # function foo() {}
            r'(?:const|let|var)\s+(\w+)\s*=\s*(?:async\s+)?\([^)]*\)\s*=>',  # const foo = () => {}
            r'async\s+function\s+(\w+)\s*\([^)]*\)',  # async function foo() {}
        ]

        functions = []
        lines = source.split('\n')

        for i, line in enumerate(lines, 1):
            for pattern in func_patterns:
                match = re.search(pattern, line)
                if match:
                    functions.append({
                        "name": match.group(1),
                        "line": i,
                        "type": "function"
                    })

        return functions

    def extract_classes(self, source: str) -> List[Dict]:
        """提取类"""

        import re

        classes = []
        lines = source.split('\n')

        for i, line in enumerate(lines, 1):
            match = re.search(r'class\s+(\w+)', line)
            if match:
                classes.append({
                    "name": match.group(1),
                    "line": i
                })

        return classes
```

## AST 可视化

### 生成 AST 可视化

```python
def ast_to_mermaid(tree: ast.AST) -> str:
    """将 AST 转换为 Mermaid 图表"""

    lines = ["graph TD"]

    node_counter = [0]

    def get_node_id(node):
        node_counter[0] += 1
        return f"Node{node_counter[0]}"

    def process_node(node, parent_id=None):
        node_id = get_node_id(node)
        node_type = type(node).__name__

        # 添加节点
        label = f"{node_type}"
        if hasattr(node, 'name'):
            label += f"<br/>({node.name})"
        elif hasattr(node, 'id'):
            label += f"<br/>({node.id})"

        lines.append(f"    {node_id}[{label}]")

        # 添加边
        if parent_id:
            lines.append(f"    {parent_id} --> {node_id}")

        # 递归处理子节点
        for child in ast.iter_child_nodes(node):
            process_node(child, node_id)

    # 从根节点开始
    process_node(tree)

    return "\n".join(lines)
```

### 使用示例

```python
# 解析代码
source = """
def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)
"""

analyzer = PythonASTAnalyzer()
tree = analyzer.parse(source)

# 提取符号
symbols = analyzer.extract_symbols()
for sym in symbols:
    print(f"{sym.type}: {sym.name} at line {sym.line}")

# 分析依赖
dep_analyzer = DependencyAnalyzer()
deps = dep_analyzer.analyze_imports(source)
print(f"Imports: {deps['all_imported_modules']}")

# 分析控制流
cf_analyzer = ControlFlowAnalyzer()
cf_info = cf_analyzer.analyze(source)
print(f"Complexity: {cf_info['functions'][0]['complexity']}")

# 生成可视化
mermaid_code = ast_to_mermaid(tree)
print(mermaid_code)
```

## AST 应用场景

### 1. 代码生成

```python
def generate_ast_from_template(template: str, variables: dict) -> ast.AST:
    """从模板生成 AST"""
    # 实现模板到 AST 的转换
    pass
```

### 2. 代码转换

```python
def convert_python_to_js(python_code: str) -> str:
    """Python 转 JavaScript"""

    # 解析 Python AST
    tree = ast.parse(python_code)

    # 遍历并转换
    js_code = []

    for node in ast.walk(tree):
        if isinstance(node, ast.FunctionDef):
            # 转换为 JS 函数
            js = f"function {node.name}({_extract_args(node.args)}) {{"
            js_code.append(js)

            # 转换函数体
            for stmt in node.body:
                js_code.append(f"  {_convert_statement(stmt)}")

            js_code.append("}")

    return "\n".join(js_code)
```

### 3. 静态分析

```python
class Linter:
    """基于 AST 的代码检查"""

    def __init__(self):
        self.rules = []

    def add_rule(self, name: str, check_func):
        """添加检查规则"""
        self.rules.append({"name": name, "check": check_func})

    def lint(self, source: str) -> List[Dict]:
        """执行检查"""
        tree = ast.parse(source)
        issues = []

        for rule in self.rules:
            result = rule["check"](tree)
            if result:
                issues.extend(result)

        return issues


# 示例规则：检测过长函数
def check_long_function(tree):
    issues = []

    for node in ast.walk(tree):
        if isinstance(node, ast.FunctionDef):
            if node.end_lineno and node.lineno:
                length = node.end_lineno - node.lineno
                if length > 100:
                    issues.append({
                        "rule": "long-function",
                        "severity": "warning",
                        "message": f"函数 {node.name} 超过 100 行 ({length} 行)",
                        "line": node.lineno
                    })

    return issues
```

## 下一步

- [01-代码切分策略](./01-代码切分策略.md) - 了解代码切分
- [02-代码Embedding](./02-代码Embedding.md) - 了解代码向量化
- [04-Code-Graph构建](./04-Code-Graph构建.md) - 了解代码图谱