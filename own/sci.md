# 科学原理：LangGraph

## 涉及的核心领域

LangGraph 的执行引擎基于以下计算机科学领域：

- **图计算（Graph Computing）**：特别是面向顶点的计算模型。
- **分布式计算理论**：Bulk Synchronous Parallel（BSP）模型。
- **Actor 模型**：并发计算中的消息驱动 actor 抽象。
- **状态机与持久化**：Checkpoint/Restore 机制，用于容错和可恢复执行。

## 原理详解

### Pregel 算法 / Bulk Synchronous Parallel (BSP) 模型

#### 问题定义

如何在一个由多个独立计算单元（节点/actors）组成的系统中，保证：
1. 每个单元可以并行执行；
2. 单元之间通过消息/状态传递进行通信；
3. 整个系统的执行是确定性的、可恢复的、可观测的。

#### 核心思想

LangGraph 的执行引擎 `Pregel` 直接借鉴了 Google 的 **Pregel** 图计算框架和 **BSP** 模型的核心思想：

> 将计算划分为一系列**全局同步的超步（supersteps）**。在每个超步中，所有活跃的 actors 并行执行，读取上一步的消息，进行本地计算，并发送消息给下一步。超步之间通过全局屏障（barrier）同步。

LangGraph 将每个超步称为一个 **step**，每个 step 包含三个阶段：

1. **Plan（计划）**：根据上一步中被更新的通道（channels），确定哪些 actors（`PregelNode`）在本 step 中需要被执行。
2. **Execute（执行）**：并行执行所有选中的 actors。每个 actor 读取其订阅的通道值，执行计算，并将结果写入目标通道。**在本阶段，所有写入对其他 actors 不可见**。
3. **Update（更新）**：当所有 actors 执行完毕后，统一将所有写入应用到通道和 Checkpoint 中。这一步完成后，通道的新值才对下一步可见。

重复上述过程，直到没有 actors 被选中执行，或达到最大 step 限制。

#### 数学表述

设图由节点集合 $V$ 和通道集合 $C$ 组成。

- 每个节点 $v \in V$ 订阅一组通道 $C_v^{in} \subseteq C$，并写出一组通道 $C_v^{out} \subseteq C$。
- 每个通道 $c \in C$ 在 step $t$ 有一个版本 $version_t(c)$ 和一个值 $value_t(c)$。
- 在 step $t$ 的 Plan 阶段，节点 $v$ 被选中执行当且仅当：
  $$\exists c \in C_v^{in} : version_t(c) > version_{seen}(v, c)$$
  即存在某个订阅通道自该节点上次执行以来被更新过。

- 在 step $t$ 的 Update 阶段，对于每个被写入的通道 $c$，应用其更新函数：
  $$value_{t+1}(c) = update_c(value_t(c), \{writes_t(v, c) \mid v \in V\})$$
  并将版本推进：
  $$version_{t+1}(c) = next_version(version_t(c))$$

#### 在代码中的体现

| 理论概念 | 代码实现 | 文件 |
|----------|----------|------|
| 超步（superstep） | `PregelLoop` 的 `step` 计数器 | `pregel/_loop.py` |
| Plan 阶段 | `prepare_next_tasks()` | `pregel/_algo.py` |
| Execute 阶段 | `PregelRunner.tick()` | `pregel/_runner.py` |
| Update 阶段 | `apply_writes()` | `pregel/_algo.py` |
| 通道版本控制 | `Checkpoint["channel_versions"]` | `checkpoint/base/__init__.py` |
| 全局屏障同步 | `FuturesDict` + `Event` 等待所有任务完成 | `pregel/_runner.py` |
| Actor | `PregelNode` | `pregel/_read.py` |
| 消息/状态传递 | `BaseChannel.update()` / `get()` | `channels/base.py` |

具体代码映射：

- **`apply_writes()`**（`pregel/_algo.py`）：
  实现 Update 阶段的核心逻辑。它收集所有 tasks 的 writes，按通道分组，调用各通道的 `update()` 方法，并更新 `checkpoint["channel_versions"]`。

- **`prepare_next_tasks()`**（`pregel/_algo.py`）：
  实现 Plan 阶段。它遍历所有 `PregelNode`，检查其订阅的通道版本是否大于该节点在 `checkpoint["versions_seen"]` 中记录的最后 seen 版本，从而决定节点是否需要在下一步执行。

- **`PregelRunner.tick()`**（`pregel/_runner.py`）：
  实现 Execute 阶段。它通过 `Submit` 接口（线程池或异步任务）并发提交所有准备好的 tasks，并通过 `FuturesDict` 等待它们全部完成，再回调 `put_writes()`。

- **`PregelLoop`**（`pregel/_loop.py`）：
   orchestrates 整个 BSP 循环，维护 `checkpoint`、`channels`、`tasks` 的状态，并在每一步调用 `prepare_next_tasks` → `runner.tick` → `apply_writes`。

### 参考文献

- Malewicz, G., et al. (2010). *Pregel: A System for Large-Scale Graph Processing*. ACM SIGMOD. https://research.google/pubs/pub37252/
- Valiant, L. G. (1990). *A Bridging Model for Parallel Computation*. Communications of the ACM. (BSP 模型原始论文)

## 与同类方法的对比

| 特性 | LangGraph (Pregel/BSP) | 传统 DAG 执行器 | 纯 Actor 模型（如 Akka） |
|------|------------------------|-----------------|--------------------------|
| 循环支持 | ✅ 原生支持 | ❌ 通常不支持 | ✅ 支持 |
| 确定性 | ✅ 全局同步保证 | ✅ 拓扑排序保证 | ⚠️ 消息顺序可能不确定 |
| 容错/恢复 | ✅ Checkpoint 每步保存 | ❌ 通常无 | ⚠️ 需额外实现 |
| 并行度 | ✅ 每步内全并行 | ⚠️ 仅拓扑并行 | ✅ 高度并行 |
| 延迟 | ⚠️ 每步有屏障开销 | ✅ 更低 | ✅ 更低 |
| 可观测性 | ✅ 每步状态完全可见 | ⚠️ 中间状态难获取 | ⚠️ 需额外追踪 |

## 局限性与假设

LangGraph 的 BSP 执行模型建立在以下假设之上：

1. **Step 级同步是可接受的**：应用能够容忍每步结束时的全局同步开销。对于需要极低延迟、流式逐 token 响应的场景，虽然 LangGraph 支持 `stream_mode="messages"`，但核心的 step 屏障仍然存在。
2. **状态是可序列化的**：当使用 Checkpoint 时，所有通道值和节点输出必须能被 `SerializerProtocol` 序列化（默认 JSON+ / msgpack）。不可序列化的对象（如打开的文件句柄、数据库连接）需要通过 `Context` 通道或 `Runtime` 注入。
3. **节点是无副作用或幂等的**：虽然框架不强制要求，但为了正确利用 Checkpoint 恢复和中断续传，节点最好是纯函数或幂等的。具有外部副作用的节点在恢复时可能导致重复执行。
4. **图结构在编译后静态**：`StateGraph.compile()` 生成静态的 `Pregel` 图。虽然支持动态路由（`Command`、条件边），但节点和通道的集合在运行时是固定的。
