# 开发指南：LangGraph

## 环境搭建

### 系统要求

- Python >= 3.10（支持 CPython 和 PyPy）
- 推荐操作系统：Linux / macOS / Windows（WSL 更佳）
- 可选：Docker（用于 PostgreSQL 集成测试）

### 依赖安装

本仓库使用 `uv` 或 `pip` 进行依赖管理。以核心库 `langgraph` 为例：

```bash
cd libs/langgraph
uv sync --all-extras
# 或
pip install -e ".[dev]"
```

由于 monorepo 中各库相互依赖（如 `langgraph` 依赖 `langgraph-checkpoint`、`langgraph-prebuilt`），推荐使用 `uv` 的 workspace 支持或分别安装 editable 依赖：

```bash
cd libs/checkpoint && uv pip install -e .
cd libs/checkpoint-sqlite && uv pip install -e .
cd libs/prebuilt && uv pip install -e .
cd libs/langgraph && uv pip install -e ".[dev]"
```

### 配置文件说明

- `pyproject.toml`：各库的配置中心，包含依赖、pytest、ruff、mypy 配置。
- `Makefile`：提供统一的 `format`、`lint`、`test` 命令。

## 本地运行

### 常用开发命令

在每个库目录下（如 `libs/langgraph/`）：

```bash
make format    # 运行 ruff format
make lint      # 运行 ruff check + mypy
make test      # 运行 pytest
TEST=tests/test_algo.py make test   # 运行指定测试文件
```

### 启动示例

LangGraph 本身是库而非独立服务。若需运行 LangGraph Server，请使用 `langgraph-cli`：

```bash
pip install langgraph-cli
langgraph up   # 启动本地开发服务器
```

## 测试

### 测试框架

- **pytest**：主测试框架
- **pytest-asyncio**：异步测试支持
- **syrupy**：快照测试
- **pytest-mock**：Mock 支持

### 运行测试

```bash
# 运行全部测试
make test

# 运行特定测试文件
pytest tests/test_algo.py -v

# 运行带覆盖率报告
pytest --cov=langgraph --cov-report=html
```

### 集成测试

部分测试需要外部服务（PostgreSQL、Redis），通过 `compose-postgres.yml` / `compose-redis.yml` 启动：

```bash
docker-compose -f libs/langgraph/tests/compose-postgres.yml up -d
pytest tests/ -k postgres
```

## 调试技巧

### 日志与追踪

- 设置 `LANGCHAIN_DEBUG=1` 可开启 LangChain 级别的详细日志。
- Pregel 执行过程中会生成 debug 事件（`stream_mode="debug"`），包含 checkpoint、任务启动/完成等信息。

### 常用调试手段

1. **使用 `stream_mode="debug"`**：
   ```python
   for event in graph.stream(input, stream_mode="debug"):
       print(event)
   ```

2. **获取状态快照**：
   ```python
   snapshot = graph.get_state(config)
   print(snapshot)
   ```

3. **LangSmith 追踪**：配置 `LANGCHAIN_TRACING_V2=true` 和 `LANGCHAIN_API_KEY`，所有执行步骤会自动上传到 LangSmith 进行可视化分析。

### 已知易踩的坑

- **修改已编译的图**：`StateGraph.add_node()` 在 `.compile()` 之后调用会打印警告，但不会影响已编译的图。需要重新 `compile()`。
- **Checkpoint 与 `thread_id`**：使用 `checkpointer` 时必须在 `config["configurable"]` 中传入 `thread_id`，否则状态无法持久化。
- **异步与同步混用**：`Pregel` 同时提供 `invoke`（同步）和 `ainvoke`（异步），在异步环境中务必使用 `ainvoke` / `astream`，避免阻塞事件循环。

## 代码规范

### 命名约定

- 类名：`PascalCase`
- 函数/变量：`snake_case`
- 私有模块/函数：前缀 `_`
- 常量：`UPPER_SNAKE_CASE`

### 目录组织约定

- 内部实现放在 `_internal/` 子包下，不保证向后兼容。
- `__init__.py` 仅导出公开 API。
- 类型定义放在 `types.py`，运行时类型放在 `typing.py`。

### 提交信息格式

未强制要求特定格式，但建议遵循常规提交规范（Conventional Commits），便于生成 Changelog。

## 贡献流程

### 分支策略

- 主分支：`main`
- 功能开发：从 `main` 切出 feature branch

### PR / Code Review 流程

1. 修改代码后，在对应库目录运行 `make format`、`make lint`、`make test`。
2. 提交 PR，CI 会自动运行测试矩阵（多 Python 版本）。
3. 维护者 review 通过后合并。

### 发版流程

- 版本号定义在各库的 `pyproject.toml` 中。
- 发布由维护者通过 GitHub Actions 自动推送到 PyPI。
