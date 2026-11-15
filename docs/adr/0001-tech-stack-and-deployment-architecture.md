# 0001: 全栈技术选型、多模型抽象与内网 Docker 部署架构

- **Status**: accepted

结合数据清洗（百至数万条 Excel/CSV/偏 JSON）、多节点 DAG **Workflow** 异步批量跑测与多模型对比调优的核心负载，本平台采用 **前后端分离单仓库架构（React Vite TS SPA + Python FastAPI + Taskiq 异步 Worker）**，以 **PostgreSQL + Redis + Docker 本地持久卷** 为基础设施，在后端进程内直接集成 **LiteLLM Python SDK**，并由 **Nginx** 作为统一 Docker Compose 入口提供 HTTP/2 SSE 实时 **Execution Trace** 推流与轻量级 **Workspace** 隔离。

## Considered Options

1. **后端语言与执行引擎**：选定 **Python (FastAPI + Taskiq + SQLAlchemy 2.0 async + Alembic + PostgreSQL `JSONB`)**，放弃纯 TypeScript 全栈（Next.js Standalone 或 Fastify + BullMQ/pg-boss）。Python 拥有明显更强的表格流式解析（`python-calamine`）、Schema 引导的偏 JSON 修复（`json_repair`）以及多模型计费生态（LiteLLM），而独立常驻 Taskiq Worker 避免了 HTTP 请求生命周期对长耗时批量跑测的限制；前后端类型契约改由 FastAPI OpenAPI Schema 通过 `openapi-typescript` + `openapi-fetch` 自动生成。
2. **多模型调用抽象**：选定 **进程内集成 LiteLLM Python SDK (`litellm.acompletion`)**，放弃额外独立部署 LiteLLM Proxy 容器。避免维护平台 PostgreSQL 与 Proxy Prisma 两套数据库，同时可直接在业务进程内精准实现“原生 `strict` JSON Schema → 降级 `json_object` → `json_repair` + Pydantic 校验”的两级结构化输出兜底、Redis 分布式按 Provider/Model 限流退避，以及“LiteLLM 默认价格/能力表 + 平台自定义覆盖”。
3. **基础设施与存储**：选定 **PostgreSQL + Redis + Docker Local Volume（代码层抽象统一存储接口预留 S3/MinIO 兼容切换）**，放弃 SQLite 单文件库（并发批量写入锁冲突）与免 Redis 纯 PostgreSQL 队列（Redis 对高并发 RPM/TPM 信号量限流与 SSE Pub/Sub 广播更直接）。
4. **前端工作台栈**：选定 **React + TypeScript (Vite SPA) + Tailwind CSS + shadcn/ui + `@xyflow/react` + TanStack Table/Query + Zustand + Monaco Editor (含 DiffEditor)**，优先满足 DAG 编排画布、大数据集虚拟表格与暗/亮色 Prompt/JSON Diff 视图的定制灵活性。
5. **内网部署、入口与安全边界**：Docker Compose 前置 **Nginx**（开启 HTTP/2 并对 SSE 路径关闭 `proxy_buffering` / `X-Accel-Buffering: no`，消除浏览器 HTTP/1.1 同域 6 连接限制）；采用轻量本地账号体系与多 **Workspace** 隔离（自带开箱可用的 `Default Workspace`），并使用 `.env` 中的服务端主密钥（`APP_ENCRYPTION_KEY`）通过 `Fernet` 对称加密持久化各 **Model Provider** API Key（支持 `.env` 预置默认配置）。

## Consequences

- **前后端契约流程**：后端修改 Pydantic Request/Response 模型后，需同步重新生成前端 `openapi-typescript` 类型定义并纳入构建检查。
- **密钥与环境管理**：部署时必须持久化生成并保管 `.env` 中的 `APP_ENCRYPTION_KEY`，密钥轮换需配合数据库密文重加密脚本；Nginx 生产/内网配置需显式保留 SSE 关闭缓冲指令。
- **异步队列与数据库演化**：任务状态与实时推流跨 PostgreSQL（持久态 **Execution Trace**）与 Redis（Taskiq Broker / Pub/Sub / 限流器），后续数据模型设计需明确断线续传（`Last-Event-ID`）与状态一致性边界。
