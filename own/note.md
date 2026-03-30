# 代码结构：LangGraph

## 顶层目录一览

| 路径 | 类型 | 用途说明 |
|------|------|----------|
| `libs/` | 源码 | 所有库代码，按 monorepo 组织 |
| `examples/` | 示例 | 已归档的示例代码（官方文档已迁移） |
| `docs/` | 文档 | 文档生成相关脚本 |
| `.github/` | 配置 | CI/CD、Issue 模板、Logo 等 |
| `dive-into-code/` | 工具 | 本仓库分析使用的 SKILL 资源 |

## 核心源码目录详解

### `libs/langgraph/langgraph/`

> 职责：LangGraph 核心框架，包含图构建、Pregel 执行引擎、通道抽象和类型定义。

| 文件/目录 | 用途 |
|-----------|------|
| `graph/` | 图构建 API（StateGraph、MessageGraph、边/分支定义） |
| `pregel/` | Pregel 执行引擎（核心运行时） |
| `channels/` | 通道实现（LastValue、Topic、BinaryOperatorAggregate 等） |
| `managed/` | Managed Value（如 RemainingSteps）生命周期管理 |
| `func/` | Functional API（`@task`、`@entrypoint`） |
| `types.py` | 核心类型定义（StreamMode、Command、RetryPolicy 等） |
| `constants.py` | 公共常量（START、END、TAG_HIDDEN） |
| `errors.py` | 异常类定义 |
| `runtime.py` | Runtime 上下文对象 |

#### `libs/langgraph/langgraph/graph/`

| 文件 | 用途 |
|------|------|
| `state.py` | `StateGraph` / `CompiledStateGraph` 定义，图构建主入口 |
| `message.py` | `MessageGraph`、`MessagesState`、`add_messages` |
| `_node.py` | 内部节点规范 `StateNodeSpec` |
| `_branch.py` | 条件分支 `BranchSpec` 实现 |

#### `libs/langgraph/langgraph/pregel/`

| 文件 | 用途 |
|------|------|
| `main.py` | `Pregel` 类和 `NodeBuilder`，执行引擎主入口 |
| `_loop.py` | `PregelLoop` / `AsyncPregelLoop` / `SyncPregelLoop`，执行循环核心 |
| `_algo.py` | Pregel 算法实现：`apply_writes`、`prepare_next_tasks`、`should_interrupt` |
| `_runner.py` | `PregelRunner`，负责任务的并发提交与结果收集 |
| `_executor.py` | 后台执行器（线程池/异步）封装 |
| `_read.py` | `PregelNode`、`ChannelRead`，节点输入读取逻辑 |
| `_write.py` | `ChannelWrite`、`ChannelWriteEntry`，节点输出写入逻辑 |
| `_io.py` | 输入映射、输出映射、通道读取辅助函数 |
| `_checkpoint.py` | Checkpoint 与 Channel 状态互转 |
| `_retry.py` | 任务重试策略实现 |
| `debug.py` | 调试信息输出（任务、写入、Checkpoint 可视化） |

#### `libs/langgraph/langgraph/channels/`

| 文件 | 用途 |
|------|------|
| `base.py` | `BaseChannel` 抽象基类 |
| `last_value.py` | `LastValue`：存储最后一个值，默认通道类型 |
| `topic.py` | `Topic`：PubSub 主题，支持累积和去重 |
| `binop.py` | `BinaryOperatorAggregate`：二元运算符聚合通道 |
| `ephemeral_value.py` | `EphemeralValue`：单步有效的临时值 |
| `any_value.py` | `AnyValue`：任意值覆盖 |
| `named_barrier_value.py` | `NamedBarrierValue`：命名屏障，用于同步 |

### `libs/prebuilt/langgraph/prebuilt/`

> 职责：高层预构建 API，降低常见智能体模式的开发门槛。

| 文件 | 用途 |
|------|------|
| `chat_agent_executor.py` | `create_react_agent`：创建 ReAct 风格智能体 |
| `tool_node.py` | `ToolNode`：在图中执行工具的节点 |
| `tool_validator.py` | `ValidationNode`：工具参数校验节点 |

### `libs/checkpoint/langgraph/checkpoint/`

> 职责：Checkpoint 持久化基础接口和内存实现。

| 文件 | 用途 |
|------|------|
| `base/__init__.py` | `BaseCheckpointSaver`、`Checkpoint`、`CheckpointTuple`、`CheckpointMetadata` |
| `memory/__init__.py` | `InMemorySaver`：内存中的 Checkpoint 存储 |
| `serde/` | 序列化/反序列化实现（JSON+、msgpack、加密支持） |

### `libs/checkpoint-sqlite/`

> 职责：基于 SQLite 的 Checkpoint 持久化实现。

| 文件 | 用途 |
|------|------|
| `langgraph/checkpoint/sqlite/__init__.py` | `SqliteSaver` |

### `libs/checkpoint-postgres/`

> 职责：基于 PostgreSQL 的 Checkpoint 持久化实现。

| 文件 | 用途 |
|------|------|
| `langgraph/checkpoint/postgres/` | `PostgresSaver`、`AsyncPostgresSaver` |

### `libs/sdk-py/langgraph_sdk/`

> 职责：Python SDK，用于与 LangGraph Server REST API 交互。

### `libs/cli/langgraph_cli/`

> 职责：LangGraph 官方 CLI 工具，支持项目初始化、构建、部署。

## 关键文件说明

### `libs/langgraph/langgraph/graph/state.py`
- **作用**：图构建的核心入口。用户通过 `StateGraph` 定义状态模式、节点、边，调用 `.compile()` 生成可执行的 `CompiledStateGraph`。
- **主要类/函数**：`StateGraph`、`CompiledStateGraph`、`add_node()`、`add_edge()`、`compile()`
- **与其他文件的关系**：编译后生成 `Pregel` 对象（`pregel/main.py`），依赖 `channels/` 和 `managed/` 创建通道。

### `libs/langgraph/langgraph/pregel/main.py`
- **作用**：Pregel 执行引擎主入口。管理 actors（节点）和 channels 的生命周期，实现 BSP 三步循环（Plan → Execute → Update）。
- **主要类/函数**：`Pregel`、`NodeBuilder`
- **与其他文件的关系**：被 `StateGraph.compile()` 调用，依赖 `_loop.py` 执行具体步骤。

### `libs/langgraph/langgraph/pregel/_loop.py`
- **作用**：Pregel 执行循环的核心实现，维护 checkpoint、状态、任务队列，协调每一步的执行。
- **主要类/函数**：`PregelLoop`、`AsyncPregelLoop`、`SyncPregelLoop`
- **与其他文件的关系**：被 `Pregel.invoke()` / `Pregel.astream()` 调用，依赖 `_algo.py` 应用写入和准备下一步任务。

### `libs/langgraph/langgraph/pregel/_algo.py`
- **作用**：Pregel 算法的纯逻辑实现，无 I/O。
- **主要类/函数**：`apply_writes()`、`prepare_next_tasks()`、`should_interrupt()`
- **与其他文件的关系**：被 `_loop.py` 调用，操作 `Checkpoint` 和 `BaseChannel`。

### `libs/langgraph/langgraph/channels/base.py`
- **作用**：定义所有通道的抽象接口。
- **主要类/函数**：`BaseChannel`（`get()`、`update()`、`checkpoint()`、`from_checkpoint()`）
- **与其他文件的关系**：所有具体通道（LastValue、Topic 等）继承自此基类。

### `libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py`
- **作用**：提供 `create_react_agent`，快速构建支持工具调用的 ReAct 智能体。
- **主要类/函数**：`create_react_agent()`
- **与其他文件的关系**：内部构建 `StateGraph`，使用 `ToolNode` 作为工具执行节点。

### `libs/prebuilt/langgraph/prebuilt/tool_node.py`
- **作用**：在 LangGraph 工作流中执行工具的节点，支持并行执行、错误处理、状态注入。
- **主要类/函数**：`ToolNode`、`InjectedState`、`InjectedStore`、`tools_condition()`
- **与其他文件的关系**：作为节点添加到 `StateGraph` 中。

### `libs/checkpoint/langgraph/checkpoint/base/__init__.py`
- **作用**：定义 Checkpoint 持久化的基础接口和数据结构。
- **主要类/函数**：`BaseCheckpointSaver`、`Checkpoint`、`CheckpointTuple`、`CheckpointMetadata`
- **与其他文件的关系**：被 `PregelLoop` 使用，具体实现由 `checkpoint-sqlite` / `checkpoint-postgres` 提供。

## 入口点

- **库公开 API 入口**：
  - `libs/langgraph/langgraph/graph/__init__.py`：`StateGraph`、`START`、`END`
  - `libs/langgraph/langgraph/pregel/__init__.py`：`Pregel`、`NodeBuilder`
  - `libs/langgraph/langgraph/func/__init__.py`：`task`、`entrypoint`
  - `libs/prebuilt/langgraph/prebuilt/__init__.py`：`create_react_agent`、`ToolNode`
- **CLI 入口**：`libs/cli/langgraph_cli/`
