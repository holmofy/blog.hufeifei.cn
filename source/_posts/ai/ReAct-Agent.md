
---
title: ReAct Agent架构设计模式
date: 2026-1-3
categories: AI
tags: 
- AI
- MCP
---

2025年结束，过去一年里，AI火爆的一批。从开年梁文峰深度求索的DeepSeek爆火，再到Cursor / Claude Code这种AI编码工具出现，真的颠覆了程序员的三观。AI终于从只会聊天的机器人，变成了能真正赋能工作的生产力工具。而这都归功于AI Agent，所以2025年也被称为AI Agent元年。

年底的时候[姚顺雨](https://ysymyth.github.io/)入职腾讯的事儿爆火了，回国前他写了篇《[The Second Half](https://ysymyth.github.io/The-Second-Half/)》(中文版：[AI进入下半场](https://zhuanlan.zhihu.com/p/1896310535865233616))。

<img width="1536" height="1024" alt="姚顺雨论文集" src="https://github.com/user-attachments/assets/3a09d8f6-6710-4fb9-bd13-c52053565be5" />

看了一下他的几篇论文，[ToT思维树: 利用大型语言模型进行有意识的问题解决](https://arxiv.org/abs/2305.10601)和[ReAct Agent架构](https://arxiv.org/abs/2210.03629)（中文版：[LLM Agent 的开山之作](https://zhuanlan.zhihu.com/p/1986190160241644560)）最具影响力。其中ReAct是Prompt工程时代的“分水岭论文”，也是Agent时代的“理论起点”。

这篇文章我记录一下我对AI Agent的一些思考。

> **AI Agent = LLM 作为决策器（Planner） + 工具执行循环（Act Loop）**

## ReAct与知行合一

现在的大模型更像是一个知识渊博的学者，他读了全世界文档、书籍、论文，但是从未下过地干过活，没有真正debug过线上故障，就像个博士学历的新兵蛋子。

即使是 CoT / ToT： 推理也只发生在 token 空间，没有外部环境的感知，不是真实践。

![ReAct架构图](https://i-blog.csdnimg.cn/direct/013fb3febf17434ba0e7222b2e5fe59b.png)

ReAct就是结合Reasoning和Action，将推理的内容转化成可实践的任务，再根据任务执行后环境反馈的错误再进行反思，如此循环最终达到目标。

```
生成提示词 → 调用 AI → 解析响应 → 执行工具 → 重复直到完成
```

稍微工程化成这样：

```
1. 构造上下文 / 提示词（Prompt + State）
2. 调用 LLM（决策 / 生成下一步行动）
3. 解析 LLM 输出（结构化 Action / Tool Call）
4. 执行工具（代码 / API / 系统操作）
5. 更新状态（Memory / Observation）
6. 判断是否完成，否则回到 1
```

这就是 **Agent Loop（ReAct / Tool-Use Loop）**。

真实 Agent 一定比这几步复杂，这只是 **“能跑起来的最小 Agent”**，工业级通常会加这些层：

### 1. 状态（State / Memory）

* short-term memory（当前任务上下文）
* long-term memory（历史、向量库）
* working memory（中间结果）

否则 Agent 会“失忆”。

### 2. 约束与控制（Guardrail）

* 最大循环次数（防止死循环）
* 工具白名单
* 输出 Schema 校验
* 成本 / token 限制

### 3. 失败处理（Recovery）

* Tool 执行失败 → 反馈给 LLM
* 解析失败 → 让 LLM 重试 / 修正格式
* 结果不满足目标 → 自我反思（self-critique）

### 🔹 4. 终止条件（Done / Stop）

终止不只是：

* “LLM 说完成了”

而是：

* 达成目标条件
* 没有可执行 Action
* 达到最大迭代
* 置信度 / 评分达标

## 这个抽象，本质就是一个「状态机」

从工程角度看，**Agent = 一个可控的状态机**：

```
┌────────┐
│ Prompt │
└───┬────┘
    ↓
┌────────┐
│  LLM   │  ← 决策器
└───┬────┘
    ↓
┌────────┐
│ Parser │  ← 协议层
└───┬────┘
    ↓
┌────────┐
│ Tools  │  ← 世界接口
└───┬────┘
    ↓
┌────────┐
│ State  │  ← 记忆 / 上下文
└────────┘
    ↑
    └──── loop
```

LangGraph/AutoGen/CrewAI/Spring AI Agent等AI Agent框架**本质都是在管这个循环**。

## 工程化落地：AI Agent框架

> **ReAct 循环本身很简单，Agent框架不是为“能跑”而生的，而是为“能长期稳定地跑在真实世界”而生的。**

最小 ReAct 循环真的很简单，最简版 ReAct，大概就这么点逻辑：

```text
while not done:
  prompt = build_prompt(state)
  response = call_llm(prompt)
  action = parse(response)
  observation = run_tool(action)
  state.update(observation)
```


### 模型输出不稳定（这是第一个地狱）

你以为 LLM 会乖乖输出：

```json
{ "tool": "search", "args": { "q": "xxx" } }
```

现实是：

* JSON 少个括号
* 字段拼错
* 输出半截自然语言
* tool name 拼错

**框架做的第一件事：**

* schema 校验
* 自动重试
* self-heal prompt
* function calling 兜底

> 否则你 80% 的代码都在 `try / catch + retry`

### 无限循环（新手 100% 会踩）

LLM 很擅长：

* “我再试试”
* “换个方式”
* “继续思考”

结果就是：

```
Thought → Action → Observation
Thought → Action → Observation
Thought → Action → Observation
（永不结束）
```

框架提供：

* max_steps
* stop condition
* budget / token 上限
* 失败终止策略

---

### 一旦任务变复杂，你的 while 直接崩

简单任务：

> “查天气”

真实任务：

* 先拆任务
* 子任务并行
* 中途失败回滚
* 依赖前序结果

你会从：

```rust
while !done {}
```

进化到：

```text
DAG / 状态机 / 有向图
```

**这就是 LangGraph 存在的原因。**

### 你需要“记忆”，而不是无限 prompt

新手常见做法：

* 把所有历史拼进 prompt

结果：

* token 爆炸
* 成本失控
* 模型开始胡说

框架提供：

* short-term memory
* long-term memory（向量库）
* summarization
* selective recall

### 可观测性 & Debug（这是工程分水岭）

你一定会遇到：

> “这次为什么不调用工具了？”

如果没有框架：

* 看 prompt 猜
* 重放成本极高

框架能：

* trace 每一步 Thought / Action
* replay agent
* 打 log、指标、耗时

### 安全 & 可控（上线必须）

现实世界里你不能让 LLM：

* 随便 exec
* 调未知 API
* 访问不该访问的数据

框架提供：

* tool 白名单
* sandbox
* permission
* policy enforcement
