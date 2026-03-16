# 04-Code-Graph构建

Code Graph（代码图谱）是代码的结构化知识表示，捕获了代码元素之间的各种关系。本文档详细介绍代码关系图谱的构建方法和应用场景。

## Code Graph 概述

### 什么是 Code Graph

Code Graph 是代码的元素（函数、类、变量）及其关系（调用、导入、继承）的图结构表示。

```
┌─────────────────────────────────────────────────────────────────┐
│                       Code Graph 示例                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐                        ┌──────────────┐     │
│  │   main.py    │                        │   utils.py    │     │
│  ├──────────────┤                        ├──────────────┤     │
│  │  main()     │ ────── calls ──────→   │  parse()     │     │
│  │  process()  │ ────── calls ──────→   │  validate()  │     │
│  │  config     │ ────── uses ───────→   │              │     │
│  └──────────────┘                        └──────────────┘     │
│       │                                        │              │
│       │ imports                                │ defines      │
│       ↓                                        ↓              │
│  ┌──────────────┐                        ┌──────────────┐     │
│  │   api.py     │  ←─── imports ────    │  models.py   │     │
│  ├──────────────┤                        ├──────────────┤     │
│  │  fetch()    │ ──── calls ────→       │  User        │     │
│  └──────────────┘                        └──────────────┘     │
│                                                                  │
│  图关系类型:                                                    │
│  - calls: 函数调用                                              │
│  - defines: 定义                                                │
│  - imports: 导入                                                │
│  - uses: 使用/引用                                             │
│  - inherits: 继承                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 代码图 vs 传统索引

| 特性 | 传统文本索引 | 代码图谱 |
|------|-------------|----------|
| 搜索方式 | 关键词匹配 | 关系遍历 |
| 上下文 | 无 | 完整调用链 |
| 推理能力 | 无 | 支持 |
| 构建成本 | 低 | 高 |
| 查询速度 | 快 | 中 |

## 图结构设计

### 节点类型

```python
from dataclasses import dataclass, field
from typing import List, Optional
from enum import Enum

class NodeType(Enum):
    """节点类型"""
    FILE = "file"
    FUNCTION = "function"
    CLASS = "class"
    VARIABLE = "variable"
    IMPORT = "import"
    MODULE = "module"

class EdgeType(Enum):
    """边类型"""
    DEFINES = "defines"        # 定义
    CALLS = "calls"           # 调用
    IMPORTS = "imports"        # 导入
    USES = "uses"              # 使用
    INHERITS = "inherits"      # 继承
    CONTAINS = "contains"      # 包含

@dataclass
class CodeNode:
    """代码节点"""
    id: str
    node_type: NodeType
    name: str
    file: str
    line_start: int = 0
    line_end: int = 0
    signature: str = ""        # 函数签名
    docstring: str = ""
    metadata: dict = field(default_factory=dict)

@dataclass
class CodeEdge:
    """代码边"""
    source: str          # 源节点 ID
    target: str          # 目标节点 ID
    edge_type: EdgeType
    metadata: dict = field(default_factory=dict)
```

### 图数据库构建

```python
class CodeGraphBuilder:
    """代码图构建器"""

    def __init__(self):
        self.nodes: dict[str, CodeNode] = {}
        self.edges: List[CodeEdge] = []

    def add_file(self, file_path: str) -> str:
        """添加文件节点"""

        node_id = f"file:{file_path}"

        self.nodes[node_id] = CodeNode(
            id=node_id,
            node_type=NodeType.FILE,
            name=file_path.split("/")[-1],
            file=file_path
        )

        return node_id

    def add_function(self, file_path: str, name: str,
                    line_start: int, line_end: int,
                    signature: str = "") -> str:
        """添加函数节点"""

        node_id = f"func:{file_path}:{name}"

        self.nodes[node_id] = CodeNode(
            id=node_id,
            node_type=NodeType.FUNCTION,
            name=name,
            file=file_path,
            line_start=line_start,
            line_end=line_end,
            signature=signature
        )

        # 添加到文件的边
        file_node_id = f"file:{file_path}"
        self.add_edge(file_node_id, node_id, EdgeType.CONTAINS)

        return node_id

    def add_class(self, file_path: str, name: str,
                  line_start: int, line_end: int) -> str:
        """添加类节点"""

        node_id = f"class:{file_path}:{name}"

        self.nodes[node_id] = CodeNode(
            id=node_id,
            node_type=NodeType.CLASS,
            name=name,
            file=file_path,
            line_start=line_start,
            line_end=line_end
        )

        # 添加到文件的边
        file_node_id = f"file:{file_path}"
        self.add_edge(file_node_id, node_id, EdgeType.CONTAINS)

        return node_id

    def add_edge(self, source: str, target: str, edge_type: EdgeType):
        """添加边"""

        self.edges.append(CodeEdge(
            source=source,
            target=target,
            edge_type=edge_type
        ))
```

## 关系提取

### 1. 调用关系提取

```python
class CallRelationExtractor:
    """调用关系提取器"""

    def extract_from_file(self, file_path: str, source: str) -> List[CodeEdge]:
        """从文件中提取调用关系"""

        import ast

        tree = ast.parse(source)
        edges = []

        # 收集所有函数定义
        functions = {}
        for node in ast.walk(tree):
            if isinstance(node, ast.FunctionDef):
                functions[node.name] = f"func:{file_path}:{node.name}"

        # 收集所有调用
        for node in ast.walk(tree):
            if isinstance(node, ast.Call):
                if isinstance(node.func, ast.Name):
                    func_name = node.func.id

                    # 查找定义的函数
                    if func_name in functions:
                        # 需要确定调用者的位置，这里简化处理
                        edges.append(CodeEdge(
                            source="",  # 需要上下文
                            target=functions[func_name],
                            edge_type=EdgeType.CALLS
                        ))

        return edges

    def extract_call_chain(self, graph, start_func: str,
                          max_depth: int = 3) -> List[str]:
        """提取调用链"""

        visited = set()
        result = []

        def traverse(node_id, depth):
            if depth > max_depth or node_id in visited:
                return

            visited.add(node_id)

            # 找到从这个节点出发的所有调用边
            for edge in graph.edges:
                if edge.source == node_id and edge.edge_type == EdgeType.CALLS:
                    result.append(edge.target)
                    traverse(edge.target, depth + 1)

        traverse(start_func, 0)
        return result
```

### 2. 导入关系提取

```python
class ImportRelationExtractor:
    """导入关系提取器"""

    def extract_imports(self, file_path: str, source: str) -> List[dict]:
        """提取导入关系"""

        import ast

        tree = ast.parse(source)
        imports = []

        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for alias in node.names:
                    imports.append({
                        "type": "import",
                        "module": alias.name,
                        "alias": alias.asname,
                        "file": file_path,
                        "line": node.lineno
                    })

            elif isinstance(node, ast.ImportFrom):
                for alias in node.names:
                    imports.append({
                        "type": "from_import",
                        "module": node.module,
                        "name": alias.name,
                        "alias": alias.asname,
                        "file": file_path,
                        "line": node.lineno
                    })

        return imports

    def build_import_graph(self, import_list: List[dict],
                           module_map: dict) -> List[CodeEdge]:
        """构建导入图"""

        edges = []

        for imp in import_list:
            source = f"file:{imp['file']}"

            # 查找目标模块
            module_name = imp.get("module") or imp.get("name")
            target = module_map.get(module_name)

            if target:
                edges.append(CodeEdge(
                    source=source,
                    target=target,
                    edge_type=EdgeType.IMPORTS
                ))

        return edges
```

### 3. 数据流分析

```python
class DataFlowAnalyzer:
    """数据流分析器"""

    def analyze_variable_usage(self, source: str,
                              var_name: str) -> List[dict]:
        """分析变量使用"""

        import ast

        tree = ast.parse(source)
        usages = []

        for node in ast.walk(tree):
            # 变量赋值
            if isinstance(node, ast.Assign):
                for target in node.targets:
                    if isinstance(target, ast.Name) and target.id == var_name:
                        usages.append({
                            "type": "assign",
                            "line": node.lineno,
                            "value": ast.unparse(node.value)
                        })

            # 变量使用
            elif isinstance(node, ast.Name):
                if node.id == var_name:
                    usages.append({
                        "type": "use",
                        "line": node.lineno,
                        "ctx": type(node.ctx).__name__
                    })

        return usages

    def trace_data_flow(self, graph: CodeGraph,
                       var_node: str) -> List[str]:
        """追踪数据流"""

        # 从变量定义追踪到使用
        # 简化版：只追踪定义
        results = []

        for edge in graph.edges:
            if edge.target == var_node:
                results.append(edge.source)

        return results
```

## 图数据库存储

### 使用 NetworkX

```python
import networkx as nx

class NetworkXGraphStore:
    """NetworkX 图存储"""

    def __init__(self):
        self.graph = nx.DiGraph()

    def add_node(self, node: CodeNode):
        """添加节点"""
        self.graph.add_node(
            node.id,
            node_type=node.node_type.value,
            name=node.name,
            file=node.file,
            line_start=node.line_start,
            line_end=node.line_end,
            signature=node.signature
        )

    def add_edge(self, edge: CodeEdge):
        """添加边"""
        self.graph.add_edge(
            edge.source,
            edge.target,
            edge_type=edge.edge_type.value
        )

    def get_callers(self, node_id: str) -> List[str]:
        """获取调用者"""
        return list(self.graph.predecessors(node_id))

    def get_callees(self, node_id: str) -> List[str]:
        """获取被调用者"""
        return list(self.graph.successors(node_id))

    def find_path(self, start: str, end: str) -> List[str]:
        """查找路径"""
        try:
            return nx.shortest_path(self.graph, start, end)
        except nx.NetworkXNoPath:
            return []

    def get_subgraph(self, node_id: str, depth: int = 2) -> nx.DiGraph:
        """获取子图"""
        return nx.ego_graph(self.graph, node_id, radius=depth)
```

### 使用 Neo4j（可选）

```python
class Neo4jGraphStore:
    """Neo4j 图数据库存储"""

    def __init__(self, uri: str, user: str, password: str):
        from neo4j import GraphDatabase
        self.driver = GraphDatabase.driver(uri, auth=(user, password))

    def add_node(self, node: CodeNode):
        """添加节点"""
        query = """
        MERGE (n:CodeNode {id: $id})
        SET n.type = $type, n.name = $name, n.file = $file
        """
        with self.driver.session() as session:
            session.run(query, id=node.id, type=node.node_type.value,
                       name=node.name, file=node.file)

    def find_function(self, name: str) -> List[dict]:
        """查找函数"""
        query = """
        MATCH (n:CodeNode {type: 'function', name: $name})
        RETURN n
        """
        with self.driver.session() as session:
            result = session.run(query, name=name)
            return [dict(record["n"]) for record in result]

    def find_call_chain(self, start_func: str, end_func: str) -> List[str]:
        """查找调用链"""
        query = """
        MATCH path = (start:CodeNode {name: $start})-[:calls*]->(end:CodeNode {name: $end})
        RETURN path
        LIMIT 1
        """
        with self.driver.session() as session:
            result = session.run(query, start=start_func, end=end_func)
            # 处理结果
            return []
```

## 代码图应用

### 1. 影响分析

```python
class ImpactAnalyzer:
    """代码影响分析器"""

    def __init__(self, graph_store):
        self.graph = graph_store

    def analyze_change_impact(self, file_path: str) -> dict:
        """分析修改影响"""

        file_node = f"file:{file_path}"

        # 找出所有依赖这个文件的文件
        dependents = self._find_dependents(file_node)

        # 找出所有可能受影响的函数
        affected_functions = set()
        for dep in dependents:
            # 查找依赖文件中定义的函数
            funcs = self._get_defined_functions(dep)
            affected_functions.update(funcs)

        return {
            "changed_file": file_path,
            "dependent_files": list(dependents),
            "affected_functions": list(affected_functions),
            "risk_level": self._calculate_risk(len(dependents), len(affected_functions))
        }

    def _find_dependents(self, node_id: str) -> set:
        """查找依赖者"""
        # 反向遍历导入边
        dependents = set()
        stack = [node_id]

        while stack:
            current = stack.pop()

            for edge in self.graph.edges:
                if edge.target == current and edge.edge_type == EdgeType.IMPORTS:
                    source = edge.source
                    if source not in dependents:
                        dependents.add(source)
                        stack.append(source)

        return dependents

    def _calculate_risk(self, num_dependents, num_affected) -> str:
        """计算风险等级"""
        if num_dependents > 10 or num_affected > 20:
            return "high"
        elif num_dependents > 5 or num_affected > 10:
            return "medium"
        return "low"
```

### 2. 代码搜索增强

```python
class GraphEnhancedSearch:
    """图增强的代码搜索"""

    def __init__(self, graph_store, vector_store):
        self.graph = graph_store
        self.vector = vector_store

    def search_with_context(self, query: str, top_k: int = 5) -> List[dict]:
        """带上下文的搜索"""

        # 1. 向量搜索
        vector_results = self.vector.search(query, top_k * 2)

        # 2. 图关系增强
        enhanced_results = []

        for result in vector_results:
            node_id = result.get("id")

            # 添加调用上下文
            callers = self.graph.get_callers(node_id)
            callees = self.graph.get_callees(node_id)

            result["callers"] = callers[:3]
            result["callees"] = callees[:3]

            # 计算重要性分数
            importance = self._calculate_importance(
                len(callers),
                len(callees),
                result.get("distance", 0)
            )
            result["importance"] = importance

            enhanced_results.append(result)

        # 3. 按重要性排序
        enhanced_results.sort(key=lambda x: x["importance"], reverse=True)

        return enhanced_results[:top_k]

    def _calculate_importance(self, num_callers, num_callees, distance) -> float:
        """计算重要性"""
        # 更多调用者 = 越重要
        caller_score = min(num_callers / 10, 1.0)
        # 越近的向量 = 越相关
        relevance_score = 1 - min(distance, 1.0)

        return caller_score * 0.6 + relevance_score * 0.4
```

### 3. 重构建议

```python
class RefactoringAdvisor:
    """重构建议器"""

    def suggest_refactoring(self, graph) -> List[dict]:
        """提供重构建议"""

        suggestions = []

        # 1. 检测循环依赖
        suggestions.extend(self._find_circular_deps(graph))

        # 2. 检测过深调用
        suggestions.extend(self._find_deep_calls(graph))

        # 3. 检测单一职责违反
        suggestions.extend(self._find_srp_violations(graph))

        return suggestions

    def _find_circular_deps(self, graph) -> List[dict]:
        """查找循环依赖"""
        import networkx as nx

        # 转换为无向图检测环
        undirected = graph.to_undirected()
        cycles = list(nx.simple_cycles(graph))

        return [
            {
                "type": "circular_dependency",
                "severity": "high",
                "message": f"循环依赖: {' -> '.join(cycle)}",
                "cycle": cycle
            }
            for cycle in cycles if cycle
        ]

    def _find_deep_calls(self, graph) -> List[dict]:
        """查找深度调用"""
        suggestions = []

        # 检测调用链深度
        for node in graph.nodes:
            depth = nx.shortest_path_length(graph, node)

            if depth > 5:
                suggestions.append({
                    "type": "deep_call_chain",
                    "severity": "medium",
                    "message": f"函数 {node} 调用链过深: {depth}",
                    "node": node
                })

        return suggestions
```

## 下一步

- [01-代码切分策略](./01-代码切分策略.md) - 了解代码切分
- [02-代码Embedding](./02-代码Embedding.md) - 了解代码向量化
- [03-AST代码分析](./03-AST代码分析.md) - 了解 AST 分析