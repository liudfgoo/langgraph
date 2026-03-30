# 关键数据流：LangGraph

## 数据流总览

| 数据流 | 功能说明 | 涉及核心模块 |
|--------|----------|--------------|
| 1 | `StateGraph` 构建与编译为 `Pregel` 执行图 | `langgraph.graph.state` |
| 2 | `Pregel.invoke()` 的 BSP 执行循环 | `langgraph.pregel.main`, `langgraph.pregel._loop`, `langgraph.pregel._algo`, `langgraph.pregel._runner` |
| 3 | `create_react_agent()` 构建的 ReAct 智能体执行流 | `langgraph.prebuilt.chat_agent_executor`, `langgraph.prebuilt.tool_node` |
| 4 | `Checkpoint` 持久化与恢复（含中断/续传） | `langgraph.pregel._loop`, `langgraph.checkpoint.base` |

---

## 数据流 1：StateGraph 构建与编译

### 触发方式

用户代码调用 `StateGraph.add_node()`、`add_edge()` 等方法构建图，最后调用 `StateGraph.compile()`。

### 流转路径

```text
[用户代码] → StateGraph.add_node(name, action)
         → StateGraph.add_edge(start, end)
         → StateGraph.compile(checkpointer, store)
         → CompiledStateGraph.__init__(pregel)
         → [返回可执行图]
```

### 各步骤详解

| 步骤 | 模块/函数 | 输入 | 处理逻辑 | 输出 |
|------|-----------|------|----------|------|
| 1 | `StateGraph.__init__()` | `state_schema`, `context_schema`, `input_schema`, `output_schema` | 解析 schema，调用 `_add_schema()` 为每个字段创建对应的 `BaseChannel` 或 `ManagedValueSpec` | 初始化的 `StateGraph` 实例 |
| 2 | `StateGraph.add_node()` | `name`, `action` (函数/Runnable), `input_schema` | 将 `action` 包装为 `Runnable`（`coerce_to_runnable`），创建 `StateNodeSpec` 存入 `self.nodes` | 更新后的 `StateGraph` |
| 3 | `StateGraph.add_edge()` | `start_key`, `end_key` | 将边加入 `self.edges` 或 `self.waiting_edges`（多起点边会进入 waiting_edges） | 更新后的 `StateGraph` |
| 4 | `StateGraph.compile()` | `checkpointer`, `store`, `interrupt_before/after` | 调用 `_get_channels()` 收集所有通道，将 `StateNodeSpec` 转换为 `PregelNode`，构建 `Pregel` 实例 | `CompiledStateGraph` 实例 |
| 5 | `CompiledStateGraph` | `Pregel` 实例 | 包装 `Pregel`，暴露 `invoke()`、`stream()`、`get_state()` 等用户接口 | 可执行图对象 |

### 数据格式变化

- **输入**：用户定义的 `TypedDict` / `Pydantic` schema。
- **中间**：`StateGraph` 内部将 schema 字段映射为 `dict[str, BaseChannel | ManagedValueSpec]`。
- **输出**：`Pregel` 对象，包含 `nodes: dict[str, PregelNode]` 和 `channels: dict[str, BaseChannel]`。

### 错误处理

- 节点名重复、使用保留名（`__end__`、`__start__`）会立即抛出 `ValueError`。
- 同一字段在不同 schema 中定义了冲突的 Channel 类型会抛出 `ValueError`。

---

## 数据流 2：Pregel 执行循环（invoke）

### 触发方式

用户调用 `compiled_graph.invoke(input, config)` 或 `compiled_graph.astream(input, config)`。

### 流转路径

```text
[用户调用] → Pregel.invoke(input, config)
         → PregelLoop.__init__(input, config)
         → loop.tick() [循环]
             → prepare_next_tasks()          [Plan]
             → PregelRunner.tick(tasks)      [Execute]
                 → Submit(task) → task.run() → put_writes(task_id, writes)
             → apply_writes(checkpoint, channels, tasks) [Update]
             → 保存 checkpoint
         → [循环结束] → 读取 output_channels → 返回结果
```

### 各步骤详解

| 步骤 | 模块/函数 | 输入 | 处理逻辑 | 输出 |
|------|-----------|------|----------|------|
| 1 | `Pregel.invoke()` | `input`, `config` | 入口方法，创建 `RunnableConfig`，调用 `_ainvoke()` 或 `_invoke()` | 最终状态/输出 |
| 2 | `PregelLoop.__init__()` | `input`, `config`, `checkpointer`, `nodes`, `channels` | 初始化 checkpoint（从 saver 加载或创建空 checkpoint），设置 `step=0` | 初始化后的 Loop |
| 3 | `PregelLoop.tick()` | 当前 checkpoint、channels | **Plan**：调用 `prepare_next_tasks()`，根据 `channel_versions` 和 `versions_seen` 差异决定哪些节点需要执行 | `tasks: dict[str, PregelExecutableTask]` |
| 4 | `PregelRunner.tick()` | `tasks` | **Execute**：通过 `Submit` 并发提交任务。每个任务执行其绑定的 `Runnable`，结果通过 `put_writes()` 写入 pending writes | 任务 futures |
| 5 | `PregelLoop.put_writes()` | `task_id`, `writes` | 收集任务的 writes，去重后存入 `checkpoint_pending_writes` | 更新 pending writes |
| 6 | `apply_writes()` | `checkpoint`, `channels`, `tasks` | **Update**：将 writes 按通道分组，调用 `channel.update(vals)`，更新 `channel_versions` 和 `versions_seen` | `updated_channels: set[str]` |
| 7 | Checkpoint 保存 | `checkpoint`, `config` | 调用 `checkpointer.put()` 和 `checkpointer.put_writes()` 持久化状态 | 持久化的 `CheckpointTuple` |
| 8 | 输出读取 | `output_channels` | 调用 `read_channels(channels, output_keys)` 提取最终结果 | 用户可见的输出 |

### 数据格式变化

- **输入**：用户原始输入（如 `dict` 或 `list[BaseMessage]`）。
- **Plan**：`Checkpoint`（含 `channel_values`、`channel_versions`、`versions_seen`）。
- **Execute**：每个 task 的输入是订阅通道的当前值（`dict`），输出是 `writes: list[tuple[str, Any]]`。
- **Update**：writes 被转换为通道的 `update()` 调用，更新 `channel_values`。
- **输出**：从 `output_channels` 读取的最终值。

### 错误处理

- 节点执行异常：在 `PregelRunner.tick()` 中捕获，根据 `RetryPolicy` 进行重试（`run_with_retry()` / `arun_with_retry()`）。
- 重试耗尽后：异常会向上传播，中断当前执行。若配置了 `checkpointer`，上一步的 checkpoint 已保存，可从中恢复。
- 最大步数限制：达到 `recursion_limit` 时抛出 `GraphRecursionError`。

---

## 数据流 3：ReAct Agent 执行流

### 触发方式

用户调用 `create_react_agent(model, tools).invoke(messages, config)`。

### 流转路径

```text
[用户调用] → create_react_agent(model, tools, checkpointer)
         → StateGraph(MessagesState)
         → add_node("agent", model_runnable)
         → add_node("tools", ToolNode(tools))
         → add_edge(START, "agent")
         → add_conditional_edges("agent", tools_condition, {"tools": "tools", END: END})
         → add_edge("tools", "agent")
         → compile()
         → [执行]
             → "agent" 节点：调用 LLM → 若产生 tool_calls → 路由到 "tools"
             → "tools" 节点：并行执行工具 → 返回 ToolMessage → 路由回 "agent"
             → 循环直到 LLM 不再调用工具
         → 返回最终 AIMessage
```

### 各步骤详解

| 步骤 | 模块/函数 | 输入 | 处理逻辑 | 输出 |
|------|-----------|------|----------|------|
| 1 | `create_react_agent()` | `model`, `tools`, `checkpointer`, `prompt` | 构建 `StateGraph(MessagesState)`，将模型包装为可调用节点，将工具包装为 `ToolNode` | `CompiledStateGraph` |
| 2 | `chat_agent_executor._get_model()` | `model`, `tools` | 若模型未绑定工具，则调用 `model.bind_tools(tools)` | 绑定工具后的模型 Runnable |
| 3 | `StateGraph.add_node("agent", ...)` | 模型 Runnable | 节点接收 `MessagesState`，将 messages 传入模型，返回新的 `AIMessage` | 节点规范 |
| 4 | `StateGraph.add_node("tools", ...)` | `ToolNode(tools)` | 节点接收包含 `tool_calls` 的 `AIMessage`，解析并并行执行工具 | 节点规范 |
| 5 | `tools_condition()` | `MessagesState` | 检查最后一条消息是否为 `AIMessage` 且包含 `tool_calls`。若有，返回 `"tools"`；否则返回 `END` | 路由目标 |
| 6 | `ToolNode._func()` | `state`, `runtime` | 遍历 `tool_calls`，对每个调用查找对应工具，通过 `RunnableCallable` 执行，收集结果 | `list[ToolMessage]` |
| 7 | 图执行 | `messages` | 由 `PregelLoop` 驱动，在 `"agent"` 和 `"tools"` 之间循环，直到 `tools_condition` 返回 `END` | 最终消息列表 |

### 数据格式变化

- **输入**：`list[BaseMessage]`（如 `HumanMessage`）。
- **agent 节点输出**：`AIMessage`，可能包含 `tool_calls: list[ToolCall]`。
- **tools 节点输入**：`AIMessage` 中的 `tool_calls`。
- **tools 节点输出**：`list[ToolMessage]`，每个对应一个工具调用结果。
- **最终输出**：`AIMessage`，不再包含 `tool_calls`。

### 错误处理

- 工具执行失败：`ToolNode` 捕获异常，将错误信息包装为 `ToolMessage` 返回给模型，让模型有机会自我纠正。
- 无效工具名：返回包含 `"Error: {tool} is not a valid tool..."` 的 `ToolMessage`。

---

## 数据流 4：Checkpoint 持久化与恢复（含中断/续传）

### 触发方式

用户配置了 `checkpointer`（如 `InMemorySaver`），并在 `config["configurable"]["thread_id"]` 中指定了线程 ID。

### 流转路径

```text
[用户调用 invoke] → PregelLoop.__init__()
                 → checkpointer.get_tuple(config)  [加载最新 checkpoint]
                 → 若存在：恢复 channels 状态
                 → 执行循环（tick）
                 → 每步结束：checkpointer.put(config, checkpoint, metadata, new_versions)
                 → 若触发 interrupt：抛出 GraphInterrupt，保存当前状态
                 
[用户恢复] → 传入 Command(resume=...)
           → PregelLoop.__init__()
           → 加载上次保存的 checkpoint
           → 将 resume 值写入 checkpoint
           → 继续执行循环
```

### 各步骤详解

| 步骤 | 模块/函数 | 输入 | 处理逻辑 | 输出 |
|------|-----------|------|----------|------|
| 1 | `PregelLoop.__init__()` | `config`（含 `thread_id`） | 调用 `checkpointer.get_tuple(checkpoint_config)` 查询该线程的最新 checkpoint | `CheckpointTuple` 或 `None` |
| 2 | `channels_from_checkpoint()` | `specs`, `checkpoint` | 根据 checkpoint 中的 `channel_values` 重建所有 `BaseChannel` 实例 | 恢复后的 `channels` |
| 3 | 执行中保存 | `checkpoint`, `config` | 每步 Update 后，调用 `checkpointer.put()` 保存 checkpoint 元数据，调用 `put_writes()` 保存 pending writes | 持久化状态 |
| 4 | `should_interrupt()` | `checkpoint`, `interrupt_nodes`, `tasks` | 检查当前 step 是否有需要中断的节点，且该通道自上次中断后有更新 | 待中断的 tasks 列表 |
| 5 | 中断处理 | `tasks` | `PregelLoop` 状态变为 `"interrupt_before"` 或 `"interrupt_after"`，抛出 `GraphInterrupt`，但 checkpoint 已保存 | 中断异常 |
| 6 | 恢复 | `Command(resume=value)` | 用户再次调用时，`map_command()` 将 `resume` 值映射为 writes，注入到恢复的 checkpoint 中，继续执行 | 恢复执行 |

### 数据格式变化

- **Checkpoint 结构**：`Checkpoint` TypedDict，包含 `v`（格式版本）、`id`（单调递增 ID）、`ts`（时间戳）、`channel_values`（通道值）、`channel_versions`（通道版本）、`versions_seen`（节点已见版本）。
- **持久化前**：`Checkpoint` 和 `PendingWrite` 列表通过 `SerializerProtocol`（默认 `JsonPlusSerializer`）序列化为字节。
- **恢复后**：从存储后端反序列化，重建 `Checkpoint` 和 `channels`。

### 错误处理

- 序列化失败：若通道值包含不可序列化对象，会抛出序列化异常。可通过 `Context` 通道或自定义 `serde` 解决。
- 存储后端失败：由具体 `BaseCheckpointSaver` 实现处理（如数据库连接异常）。

---

## 跨数据流的共享组件

以下模块/组件被多条数据流共用：

| 共享组件 | 被哪些数据流使用 | 说明 |
|----------|------------------|------|
| `PregelLoop` | 数据流 2、4 | 执行循环的核心 orchestrator |
| `BaseChannel` / `channels` | 数据流 1、2、4 | 状态传递和持久化的载体 |
| `BaseCheckpointSaver` | 数据流 2、4 | 状态快照的持久化接口 |
| `apply_writes()` | 数据流 2、4 | 每步结束统一应用 writes |
| `prepare_next_tasks()` | 数据流 2、3、4 | 决定下一步执行哪些节点 |
| `PregelRunner` | 数据流 2、3 | 并发任务执行器 |
