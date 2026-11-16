# 0002: MVP 架构精简、单工作区边界与执行恢复策略

- **Status**: accepted

为加速 MVP 第一阶段最小可用闭环落地，本决策在沿用 [ADR-0001](0001-tech-stack-and-deployment-architecture.md) 核心全栈基座的基础上，对前端编排能力与账号隔离边界进行精简，并明确执行恢复与模型调用工程基线（部分取代 ADR-0001 中关于 DAG 画布与多账号/多工作区隔离的约定）。

## 核心决策

1. **沿用全栈与基础设施基座**：保持前后端分离单仓库与 Docker Compose 5 容器拓扑（`nginx`、`web`、`worker`、`postgres`、`redis`），即 **React + TypeScript (Vite SPA) + Tailwind CSS + shadcn/ui + TanStack Table/Query + Zustand + Monaco Editor (含 DiffEditor)**、**Python FastAPI + Taskiq (Redis Broker) + SQLAlchemy 2.0 async + Alembic + PostgreSQL (`JSONB`)**、**Docker 本地持久卷**（代码层保留存储接口抽象，不引入 MinIO/S3）、**进程内 LiteLLM Python SDK (`litellm.acompletion`)**、**FastAPI OpenAPI → `openapi-typescript` + `openapi-fetch` 强类型契约**，以及 **Nginx HTTP/2 单端口统一入口**（SSE 路径关闭代理缓冲 `X-Accel-Buffering: no`）。
2. **移除 DAG 画布依赖**：因 MVP 聚焦线性多步工作流，前端明确移除 `@xyflow/react` 依赖（线性工作流具体领域语义由后续决策票另行定稿）。
3. **单默认 Workspace 与可选全局访问口令**：取消 MVP 阶段的本地多用户注册、登录及多 **Workspace** 切换，内置开箱即用的单一默认 **Workspace**（数据表仍保留 `workspace_id` 外键作用域以平滑支持 Phase 2 扩展）。支持在 `.env` 配置可选全局访问口令：未配置时免登录直进工作台；配置后由口令校验接口下发同域 `HttpOnly` Session Cookie，供 REST API 与浏览器原生 `EventSource`（SSE）共享鉴权。
4. **Model Provider 加密、角色配置与兜底策略**：沿用 `.env` 中 `APP_ENCRYPTION_KEY` + Python `Fernet` 对称加密持久化 **Model Provider** API Key（支持 `.env` 预置默认配置）。模型配置支持标记用途角色（批量调优 / 高阶 **Prompt Review** & **Tuning Suggestion**）；结构化输出采用“优先依据 `litellm.supports_response_schema()`（支持 Provider 开关覆盖）启用 `strict: true` JSON Schema → 不支持时降级 `json_object`/Prompt 约束 + `json_repair` + Pydantic 校验”的两级兜底；按 **Model Provider** 在 Redis 维护并发信号量（默认上限 10，可覆盖）及 429/503/超时指数退避重试（默认最多 3 次）；Token 与预估费用按“用户自定义单价覆盖 → LiteLLM 内置价格表 → 无匹配仅记 Token 不阻塞”计算。
5. **SSE 断线续传与 Worker 中断恢复**：`EventSource` 携带 `Last-Event-ID` 重连时，先从 PostgreSQL 已落盘 **Execution Trace** 补发缺失事件，再接续 Redis Pub/Sub 实时增量流；Worker 异常重启自检发现滞留 `running` 的 **Workflow Run** 时，自动将其标记为 `interrupted`（避免盲目重放导致重复扣费），支持人工触发“仅重跑未完成/失败的 **Dataset Record**”断点续跑。

## Consequences

- **Phase 2 平滑演进**：保留 `workspace_id` 外键与存储抽象层，使得后续引入多团队账号隔离或切换 S3/MinIO 无需破坏核心业务表结构。
- **鉴权与 SSE 兼容性**：采用同域 `HttpOnly` Cookie 后，前端可直接使用浏览器标准 `EventSource` 与原生 `Last-Event-ID` 机制，无需自研带自定义 Header 的 fetch-SSE 客户端。
- **后续术语与原型对齐**：线性工作流对 **Workflow** / **Node** 术语的精确定义以及前端交互结构将在后续决策票中收敛并同步更新 `GLOSSARY.md`。
