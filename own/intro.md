# 项目背景：LangGraph

## 是什么

LangGraph 是一个用于构建**有状态、多角色（multi-actor）智能体应用**的低级编排框架。它基于 Google Pregel 算法和 Bulk Synchronous Parallel（BSP）模型，将应用组织为节点（actors）通过通道（channels）通信的图结构，支持长时间运行、可容错、可中断和可恢复的工作流。

## 为什么存在

在 LangGraph 出现之前，构建复杂智能体应用通常面临以下挑战：

- **状态管理困难**：传统链式（chain）或简单 DAG 执行缺乏跨步骤的共享状态机制，难以实现多轮推理和记忆。
- **容错与恢复**：长运行工作流一旦中断难以从断点恢复，无法在生产环境可靠运行。
- **人机协作（Human-in-the-loop）**：缺乏在任意执行节点暂停、审查和修改状态的机制。
- **并行与协调**：多智能体并行执行时，需要复杂的同步和消息传递机制。

LangGraph 通过引入图计算模型、通道抽象和 Checkpoint 持久化，系统性地解决了上述问题。

## 目标与设计哲学

- **低级但灵活**：提供接近执行引擎的底层控制，同时不限制上层抽象方式。
- **状态优先**：一切节点通过读写共享状态（State）进行通信，天然支持记忆和上下文。
- **持久化执行**：通过 Checkpoint 机制，每一步都可保存、恢复、重放，实现 durable execution。
- **可扩展**：支持自定义 Channel、Managed Value、Checkpoint Saver 等扩展点。
- **与 LangChain 生态无缝集成**：基于 `langchain-core` 的 Runnable 接口，可复用现有组件。

## 适用场景

| 适合使用 | 不适合使用 |
|----------|------------|
| 需要多轮对话记忆的 Chatbot | 单次调用、无状态的简单 LLM 请求 |
| 需要工具调用循环的 ReAct Agent | 纯批处理、无分支的数据管道 |
| 需要人工审核/中断的审批工作流 | 对延迟极度敏感、无需持久化的短任务 |
| 多智能体协作系统（Multi-agent） | 传统 CRUD 业务系统 |
| 需要断点续传的长运行工作流 | |

## 技术栈

- **语言**：Python（>=3.10）、JavaScript/TypeScript（SDK 部分）
- **核心依赖**：
  - `langchain-core`：Runnable 接口、消息模型、回调管理
  - `pydantic`：类型校验与模型生成
  - `xxhash`：高性能哈希用于版本控制
- **持久化**：SQLite、PostgreSQL、Redis（可选）
- **测试**：pytest、pytest-asyncio、syrupy（快照测试）

## 项目现状

- **版本**：`langgraph` 核心库当前版本 `1.1.3`（Production/Stable）
- **维护状态**：活跃维护，由 LangChain Inc. 官方团队开发
- **社区规模**：LangChain 生态核心项目之一，被 Klarna、Replit、Elastic 等公司采用
- **同类对比**：
  - 相比 CrewAI、AutoGen 等高层框架，LangGraph 更偏底层，提供对执行流程的精细控制。
  - 相比纯 LangChain 的 `LCEL`（链式表达式），LangGraph 增加了循环、状态持久化和人机协作能力。
