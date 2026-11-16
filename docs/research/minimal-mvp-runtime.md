# 极简 MVP 运行与模型接入事实核验

访问日期：2026-11-16（UTC）。范围：[核验极简 MVP 持久化跑批与模型接入的最小技术方案](https://github.com/AlexdanerZe/PromptDone/issues/62)。适用定位：单个内部工作台、有限批量、单轮重复跑测、线性 Node、规则/人工评价。以下区分事实与推论，最终取舍留给技术决策票。

## 任务生命周期

**事实。** FastAPI 的 BackgroundTasks 来自 Starlette：响应发送后在进程内执行；多个任务按序运行，异常会阻止后续任务。Uvicorn 正常退出会等待任务，但受配置的关闭超时限制。[FastAPI][fastapi]、[Starlette][starlette]、[Uvicorn][uvicorn]。Taskiq 将消息交给 Broker，由独立 Worker 消费；Redis Broker 有不同实现，不能把任一种都当作相同恢复保证。[Taskiq][taskiq]、[Redis Broker][taskiq-redis]。

**推论/取舍。** async 或 BackgroundTasks 不等于持久队列，不能承诺崩溃、强杀或重启后自动续跑。最小候选可先持久化 Workflow Run，再由单进程限并发执行，并在启动时把遗留运行态识别为中断，供人工重跑；代价是 API 重启会终止本地执行。若要求 API 重启时继续执行，可保留独立 Worker，但仍须明确消息持久化、确认、重投和重复调用策略；独立进程本身不提供“恰好一次”。

## SQLite 与 PostgreSQL

**事实。** SQLite WAL 允许读写并行，但同一时刻只有一个写者；要求同主机，不能放网络文件系统。长读事务会阻碍 checkpoint，WAL 模式也可能返回 SQLITE_BUSY。FULL 与 NORMAL 的断电持久性有差别。官方另要求修复 WAL-reset 缺陷：3.51.3+，或已回移修复的 3.44.6/3.50.7 等版本。[WAL][wal]。PostgreSQL MVCC 提供并发访问与行级锁等能力。[PostgreSQL][pg]。

**推论/取舍。** 同机、少量写者、短事务下 SQLite 是可行备选，不能仅因“跑批”就断言不可用；模型网络等待不应占住写事务。应限制并发写入，设置 busy timeout 与有限重试，核对运行时实际 SQLite 版本，并验证 checkpoint、备份恢复与断电要求。[适用范围][sqlite-use]、[busy timeout][pragma]。它减少数据库服务，却增加单机边界；保留 PostgreSQL 减少偏离既有设计。未来迁移仍需处理类型、JSON 查询、约束与数据搬迁，ORM 不使迁移免费。[SQLite 类型][types]。

## 进度传输与取消

**事实。** SSE 是服务端事件传输；重连可携带 Last-Event-ID，事件持久化与补发仍由应用实现。[HTML 标准][sse]。

**推论/边界。** 若只需进度和最终 Execution Trace，按 Run ID 轮询持久态即可支持刷新恢复查看，代价是更新延迟与查询量；不依赖 Token SSE。停止轮询/关闭页面不应取消执行。取消应先阻止后续 Node/记录发起，已发出的请求另记状态；本次资料未建立跨厂商远端取消、幂等或不计费保证。超时/中断可能是“远端已执行但本地未确认”，重跑可能重复调用并计费，须保留 attempt 与未知状态。

## 单一兼容接口与 LiteLLM

**事实。** OpenAI SDK 支持配置 base_url；但参数支持随模型变化。LiteLLM SDK 可通过 api_base 接入兼容服务、做参数适配与自定义价格登记；不支持的 OpenAI 参数默认报错，可配置丢弃。SDK 本身不要求另部署 Proxy。[OpenAI Python][sdk]、[Chat Completions][chat]、[LiteLLM 接入][compatible]、[参数][params]、[价格覆盖][cost]。

**推论/取舍。** 单一 HTTP 接口可缩小依赖和能力面，适合已确定 endpoint；LiteLLM 保留多厂商适配与计价能力，需维护依赖/适配版本。两者都应验证实际模型 ID、认证、messages、输出长度、JSON/Schema 与响应形状；“OpenAI-compatible”不能证明全兼容，也不能静默丢参数改变跑测条件。当前无真实 endpoint，尚不能完成兼容性验收。

**事实与处理建议。** usage 可以缺失；即使 OpenAI 流式返回，断流也可能丢失最终 usage。[Chat Completions][chat]。建议保存原始 usage；缺失显示“未知”，不能记为零。自定义单价按模型/币种/生效时间快照，成本区分厂商 usage 计算与本地估算，均不等同账单。SDK 当前默认对部分连接/408/409/429/5xx 错误重试两次，默认超时十分钟，可显式覆盖；应统一重试层，限制总次数与总耗时，遵守 Retry-After，鉴权/无效参数/额度问题直接暴露。[SDK][sdk]、[重试指导][limits]。

## 待人决策与验收依据

仍未知：同时使用人数、批量上限、Node 数、模型并发/配额、Trace 大小、历史保留期、刷新延迟及可接受中断损失。以上事实不能推出固定吞吐保证。技术决策须在“可识别中断、人工重跑”与“独立执行及恢复机制”之间明确承诺，并以代表批量和重启/超时/写锁场景验收。

[ADR-0001](../adr/0001-tech-stack-and-deployment-architecture.md) 仍为已接受基线：React/Vite、FastAPI、Taskiq/Redis、PostgreSQL、Nginx 与进程内 LiteLLM。若极简版采纳 SQLite、移除 Worker/Redis 或改为轮询，应新增 ADR 说明适用范围与代价；本报告不修改旧 ADR、不选择最终架构，也不推荐具体模型或列商业价格。

## 官方来源

以下均于 2026-11-16 访问；网页会更新，实施时应结合锁定的依赖版本再次核验。

[fastapi]: https://fastapi.tiangolo.com/tutorial/background-tasks/
[starlette]: https://starlette.dev/background/
[uvicorn]: https://uvicorn.dev/server-behavior/#graceful-process-shutdown
[taskiq]: https://taskiq-python.github.io/guide/architecture-overview.html
[taskiq-redis]: https://github.com/taskiq-python/taskiq-redis#readme
[wal]: https://sqlite.org/wal.html
[pg]: https://www.postgresql.org/docs/current/mvcc-intro.html
[sqlite-use]: https://sqlite.org/whentouse.html
[pragma]: https://sqlite.org/pragma.html#pragma_busy_timeout
[types]: https://sqlite.org/datatype3.html
[sse]: https://html.spec.whatwg.org/multipage/server-sent-events.html
[sdk]: https://developers.openai.com/api/reference/python
[chat]: https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create
[compatible]: https://docs.litellm.ai/docs/providers/openai_compatible
[params]: https://docs.litellm.ai/docs/completion/input
[cost]: https://docs.litellm.ai/docs/completion/token_usage#9-register_model
[limits]: https://developers.openai.com/api/docs/guides/rate-limits
