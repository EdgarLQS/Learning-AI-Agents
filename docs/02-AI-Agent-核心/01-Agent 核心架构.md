# AI Agent 核心

## 01-Agent 定义与架构

### 经典公式

```
AI Agent = LLM + Planning + Tools + Memory
```

### 工程实现

```
AI Agent =
  LLM (大脑) +
  Planner (规划) +
  Tool System (工具) +
  Memory (记忆) +
  Environment (环境接口)
```

## 02-Agent 四大核心能力

### 1. 感知 (Perception)

Agent 读取环境信息：

```
输入：
- 用户指令
- 代码文件
- 网页内容
- API 响应
- 文件内容

工具：
- API 调用
- Web Scraping
- 文件读取
```

### 2. 推理 (Reasoning)

由 LLM 完成：

```
输入："用户要求添加登录功能"

LLM 推理：
1. 需要理解现有代码结构
2. 找到合适的插入位置
3. 生成符合项目风格的代码
```

### 3. 规划 (Planning)

复杂任务需要拆解：

```
任务："重构用户认证模块"

Agent 规划：
1. 分析现有代码
2. 找到所有相关文件和依赖
3. 设计新的 API 接口
4. 逐步修改并测试
```

### 4. 行动 (Action)

Agent 执行动作：

```
行动：
- 调用工具
- 修改代码
- 运行命令
- 发送请求
```

## 03-Agent 工作循环 (ReAct)

### ReAct 模式

```
Thought → Action → Observation → Repeat
```

### 例子

```
用户：帮我修复这个 bug

Agent 执行：
1. Thought: 我需要先查看错误日志
2. Action: read_file("error.log")
3. Observation: 发现空指针异常
4. Thought: 需要查看相关代码
5. Action: search_code("NullPointerException")
6. Observation: 找到问题位置
7. Thought: 添加空值检查
8. Action: write_file(...)
9. Observation: 修改成功
```

## 04-Agent 类型

### 1. 单 Agent

```
一个 AI 完成所有任务

结构：用户 → Agent → 完成
```

### 2. 多 Agent (Multi-Agent)

```
多个 AI 分工合作

例如：
- PM Agent: 分析需求
- Dev Agent: 编写代码
- QA Agent: 测试验证
```

### 3. 专用 Agent

```
每个 Agent 负责一个领域：

- Coding Agent: 编程
- Research Agent: 信息搜集
- Data Agent: 数据分析
```

## 05-Agent vs ChatGPT

| 功能 | ChatGPT | AI Agent |
|------|---------|----------|
| 回答问题 | ✓ | ✓ |
| 执行任务 | ✗ | ✓ |
| 调用工具 | ✗ | ✓ |
| 长期记忆 | ✗ | ✓ |
| 自动规划 | ✗ | ✓ |

### 区别

```
聊天 AI：
用户问 → AI 回答 → 结束

Agent：
用户给任务 → AI 规划 → AI 执行 → 返回结果
```

## 06-真实案例

### 1. 编程 Agent

**Claude Code**
- 自动搜索代码
- 修改多个文件
- 运行测试
- 提交代码

**Cursor**
- AI IDE
- 代码补全
- 智能重构

### 2. 研究 Agent

**Perplexity**
- 搜索网页
- 总结信息
- 引用来源

### 3. 办公 Agent

**Microsoft Copilot**
- 写文档
- 做 PPT
- 分析数据

## 07-为什么 AI Agent 对程序员很重要

### 未来开发模式

```
过去：程序员写每一行代码

现在：程序员描述需求 → AI 生成代码 → 程序员审查

未来：程序员设计架构 → AI 实现 → AI 测试 → 程序员监督
```

### 程序员角色转变

```
过去：
- 编写代码
- 调试 bug
- 维护项目

未来：
- 设计架构
- 审查 AI 输出
- 处理复杂问题
- 定义 AI 行为
```

### 需要学习的关键技术

1. Memory 系统
2. 多 Agent 协作
3. 代码理解系统
4. 工具调度系统
