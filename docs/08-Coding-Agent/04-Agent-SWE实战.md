# 04-Agent-SWE实战

SWE（Software Engineering）Benchmark 是评估 Coding Agent 能力的重要基准。本文档介绍 SWE-bench 和实战案例。

## SWE-bench 概述

```
┌─────────────────────────────────────────────────────────────────┐
│                      SWE-bench 介绍                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SWE-bench 是评估 AI Agent 解决真实软件问题的基准               │
│                                                                  │
│  数据集：                                                        │
│  • 来源：真实 GitHub issues                                      │
│  • 数量： 500+ 问题                                              │
│  • 难度： 需要多步修改                                           │
│                                                                  │
│  评估指标：                                                      │
│  • Pass@K: K 次尝试内解决问题的比例                              │
│  • Resolution Rate: 成功解决问题的比例                          │
│                                                                  │
│  难度特点：                                                      │
│  • 需要理解代码库                                                │
│  • 需要定位问题                                                  │
│  • 需要生成正确的修改                                            │
│  • 可能涉及多个文件                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Agent SWE 工作流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent SWE 工作流程                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. Issue 理解                                             │  │
│  │    - 阅读问题描述                                         │  │
│  │    - 提取关键信息                                          │  │
│  │    - 理解预期行为                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2. 代码检索                                               │  │
│  │    - 搜索相关代码                                          │  │
│  │    - 查找相关文件                                          │  │
│  │    - 理解上下文                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 3. 问题定位                                               │  │
│  │    - 分析错误                                             │  │
│  │    - 定位根因                                             │  │
│  │    - 确定修改点                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 4. 修改生成                                               │  │
│  │    - 生成修改方案                                         │  │
│  │    - 生成 Patch                                           │  │
│  │    - 验证语法                                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↓                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 5. 测试验证                                               │  │
│  │    - 运行测试                                             │  │
│  │    - 验证修复                                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 实现示例

### Agent SWE 系统

```python
class AgentSWE:
    """Agent SWE 系统"""

    def __init__(self, llm, tools):
        self.llm = llm
        self.tools = tools
        self.search = CodeSearch(tools)

    async def solve_issue(self, issue: dict) -> dict:
        """解决 GitHub issue"""

        # 1. 理解问题
        issue_text = issue["body"]
        title = issue["title"]

        # 2. 搜索相关代码
        search_results = await self._search_relevant_code(title, issue_text)

        # 3. 阅读关键文件
        code_context = await self._read_key_files(search_results)

        # 4. 分析问题并生成修复
        fix = await self._generate_fix(issue, code_context)

        # 5. 应用修复
        if fix:
            await self._apply_fix(fix)

        # 6. 验证
        test_results = await self._run_tests(issue)

        return {
            "issue": issue["number"],
            "fix_applied": bool(fix),
            "tests_passed": test_results["passed"]
        }
```

### 问题理解

```python
class IssueUnderstanding:
    """Issue 理解"""

    def __init__(self, llm):
        self.llm = llm

    async def analyze(self, issue: dict) -> dict:
        """分析 Issue"""

        prompt = f"""分析以下 GitHub Issue：

标题：{issue['title']}
内容：{issue['body']}

请提取：
1. 问题类型（bug/feature/refactor）
2. 受影响的模块
3. 预期行为
4. 实际行为
5. 关键关键词（用于搜索代码）

请以 JSON 格式返回。"""

        response = await self.llm.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )

        return self._parse_response(response)
```

### 代码检索

```python
class CodeSearcher:
    """代码检索"""

    def __init__(self, tools):
        self.tools = tools

    async def search(self, keywords: list) -> list:
        """搜索相关代码"""

        results = []

        for keyword in keywords:
            # 搜索
            result = await self.tools.execute("search", {
                "query": keyword,
                "max_results": 10
            })
            results.extend(result)

        # 去重和排序
        return self._deduplicate(results)
```

### 修复生成

```python
class FixGenerator:
    """修复生成器"""

    def __init__(self, llm):
        self.llm = llm

    async def generate(self, issue: dict, code_context: dict) -> str:
        """生成修复代码"""

        prompt = f"""根据以下信息生成修复代码：

问题描述：
{issue['title']}
{issue['body']}

相关代码：
{code_context}

要求：
1. 只返回修改的部分
2. 使用 diff 格式
3. 保持代码风格一致"""

        response = await self.llm.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )

        return self._extract_patch(response)
```

## 常见案例

### 案例 1：Bug 修复

```python
# Issue: "用户头像上传失败"
# 问题：文件大小检查逻辑错误

# 分析
issue = {
    "title": "头像上传失败",
    "body": "上传大于 1MB 的头像时提示文件过大"
}

# 定位
# 在 upload.py 中，max_size = 1024 (bytes) 应该是 1024*1024

# 修复
patch = """--- a/upload.py
+++ b/upload.py
@@ -10,7 +10,7 @@
-MAX_SIZE = 1024
+MAX_SIZE = 1024 * 1024
"""
```

### 案例 2：功能实现

```python
# Issue: "添加用户收藏功能"

# 需要修改：
# 1. models/user.py - 添加收藏字段
# 2. api/favorite.py - 添加 API
# 3. frontend/components/FavoriteButton.tsx - 前端组件
```

### 案例 3：性能优化

```python
# Issue: "列表查询太慢"

# 问题：N+1 查询
# 修复：添加 eager loading
```

## 评估与优化

### 评估指标

```python
class SWEEvaluator:
    """SWE 评估器"""

    def evaluate(self, results: list, ground_truth: list) -> dict:
        """评估结果"""

        correct = 0
        for result, expected in zip(results, ground_truth):
            if self._is_correct(result, expected):
                correct += 1

        return {
            "accuracy": correct / len(results),
            "total": len(results),
            "correct": correct
        }
```

### 优化策略

```python
# 优化点：
# 1. 更好的检索 → 提高定位准确性
# 2. 更好的 Prompt → 提高修复质量
# 3. 自我反思 → 错误后重新尝试
# 4. 测试验证 → 确保修复有效
```

## 下一步

本文档是 Coding Agent 模块的最后一篇。建议复习整个学习路径，并开始实战项目。

相关内容：
- [01-Claude Code 原理](./01-Claude Code 原理.md)
- [02-Patch生成机制](./02-Patch生成机制.md)
- [03-多文件重构](./03-多文件重构.md)