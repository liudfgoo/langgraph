# 最小使用示例：LangGraph

## 快速开始

### 环境准备

```bash
pip install -U langgraph
# 如需持久化，可选安装：
pip install langgraph-checkpoint-sqlite
```

---

## 示例 1：StateGraph 基础用法

**功能说明**：构建一个简单的状态图，节点读取状态中的数值，加 1 后写回。

**前置条件**：Python >= 3.10，已安装 `langgraph`。

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    count: int

def increment_node(state: State) -> dict:
    return {"count": state["count"] + 1}

builder = StateGraph(State)
builder.add_node("increment", increment_node)
builder.add_edge(START, "increment")
builder.add_edge("increment", END)

graph = builder.compile()
result = graph.invoke({"count": 0})
print(result)
```

**预期输出**：

```python
{'count': 1}
```

**关键参数说明**：

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `state_schema` | `type[TypedDict]` | 定义图的状态结构 | 必填 |
| `input_schema` | `type` | 输入状态结构（默认同 `state_schema`） | `None` |
| `output_schema` | `type` | 输出状态结构（默认同 `state_schema`） | `None` |

---

## 示例 2：Functional API（@task + @entrypoint）

**功能说明**：使用装饰器风格定义任务和入口函数，任务可并行执行。

**前置条件**：Python >= 3.10。异步任务需要 Python >= 3.11。

```python
from langgraph.func import entrypoint, task

@task
def add_one(x: int) -> int:
    return x + 1

@entrypoint()
def workflow(numbers: list[int]) -> list[int]:
    futures = [add_one(n) for n in numbers]
    return [f.result() for f in futures]

result = workflow.invoke([1, 2, 3])
print(result)
```

**预期输出**：

```python
[2, 3, 4]
```

**关键参数说明**：

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `retry_policy` | `RetryPolicy` | 任务失败时的重试策略 | `None` |
| `cache_policy` | `CachePolicy` | 任务结果缓存策略 | `None` |

---

## 示例 3：ReAct Agent（工具调用智能体）

**功能说明**：使用 `create_react_agent` 快速构建一个支持工具调用的智能体。

**前置条件**：需要 `langchain-core` 和一个模拟/真实的聊天模型。以下示例使用假模型演示结构。

```python
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def get_weather(city: str) -> str:
    """Get weather for a city."""
    return f"It's sunny in {city}."

# 使用一个支持 tool calling 的模型，例如 ChatOpenAI
# from langchain_openai import ChatOpenAI
# model = ChatOpenAI(model="gpt-4o-mini")

# 以下为结构演示（模型需要真实替换）
# agent = create_react_agent(model, [get_weather])
# result = agent.invoke({
#     "messages": [("user", "What's the weather in Beijing?")]
# })
# print(result["messages"][-1].content)
```

**预期输出**（假设模型正确调用了工具）：

```
It's sunny in Beijing.
```

**关键参数说明**：

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `model` | `LanguageModelLike` | 支持工具调用的聊天模型 | 必填 |
| `tools` | `list[BaseTool]` | 智能体可调用的工具列表 | 必填 |
| `checkpointer` | `BaseCheckpointSaver` | 状态持久化器 | `None` |
| `prompt` | `str / SystemMessage / Callable` | 注入到对话前的系统提示 | `None` |

---

## 示例 4：Checkpoint 与 Human-in-the-loop

**功能说明**：在工作流中插入中断点，等待人工输入后恢复执行。

**前置条件**：安装 `langgraph-checkpoint-sqlite` 或直接使用 `InMemorySaver`。

```python
from langgraph.func import entrypoint, task
from langgraph.types import interrupt
from langgraph.checkpoint.memory import InMemorySaver

@task
def generate_draft(topic: str) -> str:
    return f"Draft about {topic}"

@entrypoint(checkpointer=InMemorySaver())
def review_workflow(topic: str) -> dict:
    draft = generate_draft(topic).result()
    review = interrupt({
        "question": "Please review the draft",
        "draft": draft,
    })
    return {"draft": draft, "review": review}

config = {"configurable": {"thread_id": "demo-thread"}}

# 第一次调用，会在 interrupt 处暂停
for event in review_workflow.stream("AI", config):
    print(event)

# 模拟人工审核后恢复
from langgraph.types import Command
for event in review_workflow.stream(Command(resume="Looks good!"), config):
    print(event)
```

**预期输出**：

```python
# 第一次 stream 输出到 interrupt 前
{'draft': 'Draft about AI'}

# 恢复后最终输出
{'draft': 'Draft about AI', 'review': 'Looks good!'}
```

**关键参数说明**：

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `checkpointer` | `BaseCheckpointSaver` | 启用持久化，必须配置 | `None` |
| `thread_id` | `str` | 存储在 `config["configurable"]` 中，作为状态主键 | 必填 |
| `Command(resume=...)` | `Command` | 用于从中断点恢复执行 | - |

---

## 常见使用模式

### 1. 使用 `add_messages` 管理对话历史

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]
```

`add_messages` 是一个 reducer，它会将新消息追加到历史列表中，同时支持按 ID 覆盖已有消息。

### 2. 条件分支（Conditional Edges）

```python
from langgraph.graph import END

def route(state: State) -> str:
    if state["count"] > 5:
        return END
    return "increment"

builder.add_conditional_edges("increment", route)
```

### 3. 子图（Subgraphs）

一个 `CompiledStateGraph` 本身也是 `Runnable`，可以作为节点添加到另一个 `StateGraph` 中，实现模块化组合。

---

## 常见问题

### Q1：为什么 `compile()` 后修改 `StateGraph` 不生效？

`compile()` 会生成一个独立的 `Pregel` 执行图。后续对 `StateGraph` 的修改不会影响已编译的图。需要重新调用 `compile()`。

### Q2：使用 `checkpointer` 时为什么必须传 `thread_id`？

`thread_id` 是 `BaseCheckpointSaver` 存储和检索 checkpoint 的主键。没有它，状态无法持久化，也无法实现中断恢复。

### Q3：异步环境中应该使用 `invoke` 还是 `ainvoke`？

在异步环境（如 FastAPI、`asyncio` 事件循环）中，**务必使用 `ainvoke` / `astream`**。使用同步的 `invoke` 会阻塞事件循环，导致性能问题或超时。
