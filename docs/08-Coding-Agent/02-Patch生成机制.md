# 02-Patch生成机制

Patch（补丁）是 Agent 修改代码的核心产出。本文档详细介绍 diff 格式、Patch 生成策略和实现方法。

## Patch 概述

```
┌─────────────────────────────────────────────────────────────────┐
│                       Patch 流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  原始代码                    Agent 修改                    Patch │
│  ┌──────────┐              ┌──────────┐              ┌────────┐ │
│  │ def add  │    ──────→   │ def add  │    ──────→   │ diff   │ │
│  │   a + b  │              │   a + b  │              │ -1,3   │ │
│  └──────────┘              └──────────┘              │ +1,4   │ │
│                                                   └────────┘ │
│                                                                  │
│  实际文件                    合并 Patch                    结果  │
│  ┌──────────┐              ┌──────────┐              ┌────────┐ │
│  │ def add  │   + Patch →  │ def add  │   ──────→   │ def    │ │
│  │   a + b  │              │   a + b  │              │ add(a) │ │
│  └──────────┘              └──────────┘              └────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Diff 格式

### Unified Diff 格式

```python
# 示例：修改函数
--- a/calculator.py
+++ b/calculator.py
@@ -1,5 +1,7 @@
 def add(a, b):
+    """加法函数"""
     return a + b
+
+def subtract(a, b):
+    return a - b
```

### 格式说明

```
--- a/file.py      # 原始文件
+++ b/file.py      # 修改后文件
@@ -起始行,行数 +起始行,行数 @@  # 变化的区块
 原始行             # - 表示删除
+ 新行              # + 表示新增
```

## Patch 生成策略

### 1. 精确替换

```python
class PrecisePatcher:
    """精确替换"""

    def generate_patch(self, file_path: str,
                      old_content: str, new_content: str) -> str:
        """生成精确替换的 patch"""

        import difflib

        # 使用 unified_diff 生成
        diff = difflib.unified_diff(
            old_content.splitlines(keepends=True),
            new_content.splitlines(keepends=True),
            fromfile=f'a/{file_path}',
            tofile=f'b/{file_path}',
            lineterm=''
        )

        return ''.join(diff)
```

### 2. 行级编辑

```python
class LineEditPatcher:
    """行级编辑 patch"""

    def replace_lines(self, file_path: str,
                     start_line: int, end_line: int,
                     new_content: str) -> str:
        """替换指定行"""

        # 读取原文件
        with open(file_path, 'r') as f:
            lines = f.readlines()

        # 构建新内容
        before = lines[:start_line - 1]
        after = lines[end_line:]
        new_lines = new_content.split('\n')

        # 添加换行
        new_lines = [line + '\n' for line in new_lines]

        # 合并
        result_lines = before + new_lines + after

        # 生成 patch
        return self._generate_patch(
            file_path,
            lines,
            result_lines
        )

    def insert_lines(self, file_path: str,
                    after_line: int, new_content: str) -> str:
        """插入新行"""
        # 类似实现
        pass

    def delete_lines(self, file_path: str,
                    start_line: int, end_line: int) -> str:
        """删除行"""
        # 类似实现
        pass
```

### 3. 语义编辑

```python
class SemanticPatcher:
    """语义级别的代码修改"""

    def __init__(self, llm_client):
        self.llm = llm_client

    async def generate_semantic_patch(self, file_path: str,
                                      instruction: str) -> str:
        """基于语义生成 patch"""

        # 1. 读取文件
        with open(file_path, 'r') as f:
            content = f.read()

        # 2. 让 LLM 生成修改
        prompt = f"""请修改以下代码：

文件：{file_path}
修改要求：{instruction}

原始代码：
```{content}
```

请只返回修改后的代码，不需要解释。"""

        response = await self.llm.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )

        new_content = response.choices[0].message.content

        # 3. 生成 patch
        return self._generate_patch(file_path, content, new_content)
```

## Patch 验证

```python
class PatchValidator:
    """Patch 验证器"""

    def validate(self, patch: str, file_path: str) -> dict:
        """验证 patch 有效性"""

        # 1. 语法检查
        syntax_ok = self._check_syntax(file_path)

        # 2. 语义检查
        semantic_ok = self._check_semantics(file_path)

        # 3. 冲突检查
        conflicts = self._check_conflicts(patch)

        return {
            "valid": syntax_ok and semantic_ok,
            "syntax_ok": syntax_ok,
            "semantic_ok": semantic_ok,
            "conflicts": conflicts
        }

    def _check_syntax(self, file_path: str) -> bool:
        """语法检查"""
        try:
            import ast
            with open(file_path, 'r') as f:
                ast.parse(f.read())
            return True
        except SyntaxError:
            return False
```

## 实际应用

### 应用 Patch

```python
class PatchApplier:
    """Patch 应用器"""

    def apply(self, patch: str, target_dir: str) -> dict:
        """应用 patch"""

        import subprocess

        # 使用 patch 命令应用
        result = subprocess.run(
            ['patch', '-p1'],
            input=patch,
            capture_output=True,
            text=True,
            cwd=target_dir
        )

        return {
            "success": result.returncode == 0,
            "output": result.stdout,
            "error": result.stderr
        }
```

### 回退 Patch

```python
class PatchReverter:
    """Patch 回退器"""

    def revert(self, patch: str, target_dir: str) -> dict:
        """回退 patch"""

        # 使用 patch -R
        import subprocess

        result = subprocess.run(
            ['patch', '-R', '-p1'],
            input=patch,
            capture_output=True,
            text=True,
            cwd=target_dir
        )

        return {"success": result.returncode == 0}
```

## 下一步

- [01-Claude Code 原理](./01-Claude Code 原理.md) - 了解 Claude Code 原理
- [03-多文件重构](./03-多文件重构.md) - 了解多文件重构
- [04-Agent-SWE实战](./04-Agent-SWE实战.md) - 了解实战案例