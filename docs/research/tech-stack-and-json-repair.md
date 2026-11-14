# 技术栈、畸形 JSON 修复与多模型调用生态调研

> **关联 Issue**: [AlexdanerZe/PromptDone#2](https://github.com/AlexdanerZe/PromptDone/issues/2)（Parent: [#1](https://github.com/AlexdanerZe/PromptDone/issues/1) · Blocking: [#6](https://github.com/AlexdanerZe/PromptDone/issues/6)）  
> **调研场景**: 内部团队 Web 平台（支持 Docker 部署）+ 百至数万条 Excel/CSV/偏 JSON 数据清洗生成版本化 **Dataset** + **Workflow**（多 **Node** DAG）异步批量跑测 + 多模型调用与结构化输出 + **Execution Trace** 实时观测。  
> **术语约定**: 本文严格遵循项目术语表 [`GLOSSARY.md`](../../GLOSSARY.md) 中的规范术语（**Dataset**、**Node**、**Workflow**、**Execution Trace**、**Tuning Session**、**Context Bundle**）。

---

## 1. 候选整体技术栈对比（面向 Docker 私有化部署）

内部 Prompt 调优平台的核心工作负载分为三类：**I/O 密集型的多模型并发调用**、**CPU/内存敏感的表格与偏 JSON 清洗**、以及 **低延迟的 Execution Trace 实时观测与交互前端**。

| 评估维度 | 方案 A：Next.js (App Router) 全栈 + 独立 Worker | 方案 B：React (SPA) + Python FastAPI + 异步 Worker | 方案 C：React (SPA) + Node/TS 后端 (Fastify/Hono) + Worker |
| :--- | :--- | :--- | :--- |
| **Docker 部署形态** | `output: "standalone"` 精简镜像（需手动复制 `public/` 与 `.next/static/`），但长耗时批任务仍必须拆出独立 Worker 容器（Next.js HTTP 生命周期不适合小时级批跑测） | `docker-compose` 多容器（前端静态/Nginx + FastAPI + Taskiq/ARQ Worker + Postgres + Redis），边界清晰、无超时隐患 | `docker-compose` 多容器（前端 + Fastify/Hono API + BullMQ/pg-boss Worker + DB/Redis），前后端同语言构建简单 |
| **Excel/CSV/偏 JSON 清洗生态** | 偏弱：受限于 Node.js 生态（SheetJS 社区版无 XLSX 流式读取、ExcelJS 复杂表格易丢属性），高并发大文件易触发 V8 堆内存压力 | **最强**：`python-calamine` / `polars` / `openpyxl (read_only)` + `json_repair`（支持 Schema 引导修复与 `stream_stable`） | 中等：`ExcelJS` (`WorkbookReader`) + `csv-parse` + `josdejong/jsonrepair`（无原生 JSON Schema 引导修复） |
| **多模型与结构化输出生态** | **强**：Vercel AI SDK（`generateText` + `Output.object()` + Zod + OpenTelemetry）前后端类型一体 | **强**：LiteLLM（100+ Provider 统一接口、内置价格/Token 计算与代理追踪）及 OpenAI/Anthropic/Google 官方 Python SDK | 中强：Vercel AI SDK 或官方 TS SDK，但缺少 LiteLLM 级别的统一成本表与跨供应商特性探测 |
| **Execution Trace 实时推送** | Route Handler 返回 `ReadableStream` 实现 SSE；Next.js 路由缓冲与中间件细节需额外注意 | FastAPI 0.135.0+ 原生 `fastapi.sse.EventSourceResponse`（或 `sse-starlette`）直连 Redis Pub/Sub，天然异步流式 | Fastify/Hono 原生流式响应实现 SSE，与 BullMQ `QueueEvents` 同处 Node 生态易对接 |
| **全栈开发与类型契约** | **极佳**：TypeScript 单仓共享 Zod Schema 与类型定义 | 需维护前后端契约（OpenAPI 生成 TS Client，FastAPI 自动生成 OpenAPI） | **极佳**：TypeScript 单仓共享 Zod Schema 与类型定义 |

**核心来源**：
- Next.js Standalone Docker 部署与静态资源复制要求：https://nextjs.org/docs/app/api-reference/config/next-config-js/output
- FastAPI 原生 Server-Sent Events 文档：https://fastapi.tiangolo.com/tutorial/server-sent-events/

### 小结与架构提示
1. **即使选用 Next.js 全栈，也不应把“百至数万条 Dataset 清洗与批量跑测”放进 Next.js API Route 内执行**；在 Docker 自托管环境下，长时间批处理必须由独立常驻 Worker 容器消费队列，HTTP 层仅负责提交任务与推流（参考 Next.js 部署与长任务分离模式：https://nextjs.org/docs/app/api-reference/config/next-config-js/output ）。
2. 若团队**以数据清洗准确率、偏 JSON 容错、多模型网关兼容为首要优先级**，**方案 B（React + FastAPI + 异步 Python Worker）**在工程生态上显著领先；若团队**纯 TypeScript 栈且 Dataset 规模偏向百至千级（非重度复杂 Excel）**，**方案 C（React + Fastify/Hono + BullMQ/pg-boss）**或 **Next.js BFF + 独立 TS/Python Worker** 更易统一研发栈。

---

## 2. 畸形 / 偏 JSON 修复库深度对比

数据集导入与 LLM 非严格输出阶段常出现五类典型畸形 JSON：**单引号**、**尾逗号**、**Markdown 代码块与前后杂文**、**双重转义（字符串化 JSON）**、**截断（如到达 `max_tokens` 或流式半截输出）**。

| 能力维度 | `json-repair` (Python: `mangiucugna/json_repair`) | `jsonrepair` (JS/TS: `josdejong/jsonrepair`) | `json5` (JS: `json5/json5` · Py: `dpranke/pyjson5`) |
| :--- | :--- | :--- | :--- |
| **单引号 (`{'a': 'b'}`)** | **支持**：自动转为标准双引号 JSON；中文/非拉丁字符需传 `ensure_ascii=False` 避免被转义为 `\uXXXX` | **支持**：将单引号及排版引号（如 `“...”`）转为标准双引号 | **支持**：JSON5 规范（ES5.1 子集）原生允许单引号与未加引号标识符键 |
| **尾逗号 (`[1, 2,]`)** | **支持**：自动清理对象和数组末尾多余逗号 | **支持**：自动剥离尾逗号 | **支持**：JSON5 规范原生允许单尾逗号 |
| **Markdown 包裹与前后杂文** | **支持（有边界）**：可清理常见非 JSON 字符与围栏；官方提供 `examples/repair_llm_output.py` 专门剥离 ```json 围栏与前后对话散文，复杂文本建议先做轻量正则提取 | **半支持**：官方 README 明确支持剥离 Markdown 代码块围栏（```` ```json ````），但**无法可靠识别并剔除大段前置解释性散文**，需调用前预剥离 | **不支持**：遇到 ```` ```json ```` 或前后散文直接报错 |
| **双重转义 (`"{\"a\": 1}"`)** | **部分支持 / 有局限**：支持修复未转义引号，但多层嵌套转义（如 `\\\"`）或合法反斜杠（如 LaTeX 数学公式）场景容易混淆或破坏原有转义；深层嵌套大量转义引号可能触发递归深度异常 | **原生支持**：明确支持剥离字符串化 JSON 的多余转义符（如 `{\"stringified\": \"content\"}`），也支持拼接拆分字符串（`"a" + "b"`） | **不支持**：按字面解析为普通字符串，不会解开双重转义 |
| **截断修复（不完整 JSON）** | **支持**：自动补全缺失闭合括号/引号及缺失值（补为 `""` 或 `null`）；提供 `stream_stable=True` 专门稳定流式截断输出 | **支持（带流式缓冲限制）**：可补全未闭合括号/引号；但 Node 流式接口 `jsonrepairTransform` 默认 `bufferSize=65536` 字节，单字段或空白超长会抛 `Index out of range` 错 | **不支持**：任何截断都会抛出语法错误 |
| **额外特性与典型局限** | • 默认先调 `json.loads()` 再降级修复（切勿外层重复 `try json.loads`，官方称为反模式）<br>• 支持 `return_objects=True` 直接返回 Python 对象避免二次序列化<br>• 支持 `schema=`（JSON Schema / Pydantic v2）引导修复与 `salvage` 模式<br>• `skip_json_loads=True` **不可用于已知合法 JSON**，否则可能改写原值<br>• `strict=True` 可在重复键/缺分隔符时强制报错 | • 支持 NDJSON 转数组、JSONP 剥离、MongoDB `NumberLong`/`ISODate` 剥离<br>• 原生提供 ESM/CJS/UMD 与 Node Stream (`jsonrepair/stream`)<br>• **无 Schema 引导修复能力**<br>• 截断字符串内若含冒号 `:` 等结构定界符仍可能抛出 `JSONRepairError` | • **本质是“宽松语法解析器”而非“修复器”**：不支持缺失引号补全（非标识符键）、缺失逗号/括号修复<br>• `json5.org` 明确声明不适合机器间通用数据交换<br>• 纯 Python 版 `pyjson5` 性能比 C 扩展 `json` 慢约 1,000–6,000 倍 |

**核心来源**：
- `mangiucugna/json_repair` 官方仓库与文档（`skip_json_loads`、`return_objects`、`stream_stable`、`strict`、Pydantic Schema 引导、`ensure_ascii=False`）：https://github.com/mangiucugna/json_repair 、https://mangiucugna.github.io/json_repair/ 、https://github.com/mangiucugna/json_repair/blob/main/examples/repair_llm_output.py
- `josdejong/jsonrepair` 官方仓库（能力列表、Markdown 围栏、双重转义剥离、`jsonrepairTransform` `bufferSize` 限制）：https://github.com/josdejong/jsonrepair
- JSON5 官方规范与仓库：`https://json5.org/` 、`https://github.com/json5/json5` 、Python `pyjson5` 性能与限制说明：https://pypi.org/project/json5/ 、https://github.com/dpranke/pyjson5

### 工程选型结论
- **不建议将 `json5` 作为清洗主力**：它仅能覆盖“单引号 + 尾逗号 + 注释 + 无引号键”，对 LLM 最常见的 Markdown 围栏、双重转义、截断输出完全无能为力，且 `pyjson5` 性能极差。
- **Python 栈首选 `json_repair`（结合 `orjson` 快速路径与前置代码围栏剥离）**：建议对 Dataset 入库与 Node 输出解析采用「标准库 / `orjson` 先行解析 → 失败时剥离 Markdown 围栏 → `json_repair.repair_json(..., schema=..., return_objects=True, ensure_ascii=False)`」策略，兼顾速度与结构修复能力。
- **TS 栈首选 `josdejong/jsonrepair`**：清洗百至数万条记录时，先用正则提取围栏内文本，再调用 `jsonrepair()` + `JSON.parse()` + `Zod.safeParse()`；若遇到“JSON 字符串套 JSON 字符串”（双重转义），可对解析出的字符串结果进行一次二次判断与解包。

---

## 3. Excel / CSV 流式解析与大单元格兼容对比

针对“百至数万条”记录且包含长 Prompt、多轮对话文本或大段 JSON 的 **大单元格（Large Cell）** 场景，内存控制与字段长度上限是核心考量：

| 语言栈 | 库名称 | 流式读取能力 | 大单元格 / 内存与性能表现 | 关键限制与注意事项 |
| :--- | :--- | :--- | :--- | :--- |
| **Python** | **`python-calamine`** | 按工作表迭代读取（Rust `calamine` 绑定，内存占用低） | 支持 `.xlsx`/`.xls`/`.xlsb`/`.ods`；Rust 底层解析速度较 `openpyxl` 快 5–20 倍 | 专注于只读数据提取，不处理复杂图表/写回样式 |
| **Python** | **`openpyxl`** | 支持 `load_workbook(..., read_only=True)` 惰性流式读取 | 默认模式会将整张表加载为内存 DOM（可达文件体积 ~50x）；`read_only=True` 可维持近似常量内存 | `read_only=True` 返回 `IterableWorksheet`（`ReadOnlyCell`），不解析图表/格式且依赖文件预设维度；**必须显式调用 `wb.close()`** |
| **Python** | **`pandas`** | `read_csv(..., chunksize=N)` 原生流式分块；**`read_excel` 无 `chunksize` 参数** | Pandas 2.2+ 起 `read_excel(..., engine="calamine")` 可显著提速 | `read_excel` 无法原生流式分块（需先转 CSV/Parquet 或用 `skiprows`/`nrows` 模拟分块） |
| **Python** | **`polars`** | `scan_csv` 惰性流式扫描；`read_excel` 默认使用 `calamine` 引擎 | 内存映射与多线程列式处理，数万行级清洗极快 | `read_excel` 本质仍一次性读入表数据（底层为 `fastexcel`/`calamine`） |
| **Python** | **标准库 `csv`** | 原生逐行迭代器流式读取，零内存膨胀 | 极快，适合配合生成器清洗逐行入库 | **大单元格致命陷阱**：默认 `csv.field_size_limit()` 仅为 **131,072 字符（128 KB）**，含长上下文的单元格会直接抛 `_csv.Error`，入库前必须安全调高上限 |
| **JS/TS** | **`ExcelJS`** | **原生免费流式读取** `ExcelJS.stream.xlsx.WorkbookReader` | 可在 Node.js 逐行/逐单元格消费大 `.xlsx`，避免 V8 `heap out of memory`；MIT 协议 | 仅主要针对 `.xlsx` 与 `.csv`（不支持旧版 `.xls`/`.xlsb`）；流式读写器对格式严格，复杂数据验证/跨表引用易丢失 |
| **JS/TS** | **`SheetJS` (`xlsx`)** | **社区版无原生 XLSX 流式读取**（`XLSX.read` 全量入内存） | 支持 20+ 格式；可通过 `dense: true`、`cellStyles: false` 降低内存 | **发行与安全隐患**：npm 官方包停留在 `0.18.5`（2022），新版需通过 `https://cdn.sheetjs.com/...tgz` 安装；写样式等功能属商业 Pro 版 |
| **JS/TS** | **`PapaParse`** | 支持 `step`（逐行）与 `chunk`（逐块）流式解析，支持 `worker: true` | 浏览器端与 Node 均可流式处理超大 CSV 与大单元格 | 开启 `worker: true` 后**不可使用 `pause()` 与 `resume()`** 控制流 |
| **JS/TS** | **`csv-parse`** | 原生 Node.js `Transform` 流，支持 `for await (const record of parser)` | 背压（Backpressure）控制自然，适合服务端流式导入百万字符级大单元格 | 需正确配置 `relax_quotes` / `bom: true` 应对导出自 Excel 的 UTF-8 BOM CSV |

**核心来源**：
- `openpyxl` `read_only=True` 优化模式文档：https://openpyxl.readthedocs.io/en/stable/optimized.html
- `python-calamine` 官方仓库与 PyPI：https://github.com/dimastbk/python-calamine 、https://pypi.org/project/python-calamine/
- `Polars` `read_excel` 与 `calamine` 引擎文档：https://docs.pola.rs/api/python/stable/reference/api/polars.read_excel.html
- `Pandas` `read_csv` (`chunksize`) 与 `calamine`：https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html
- Python 标准库 `csv.field_size_limit` 默认 131072 限制：https://docs.python.org/3/library/csv.html
- `ExcelJS` 官方仓库（`WorkbookReader`）：https://github.com/exceljs/exceljs
- `SheetJS` Node.js 安装与 `cdn.sheetjs.com` 迁移说明：https://docs.sheetjs.com/docs/getting-started/installation/nodejs/
- `PapaParse` 官方文档（`step`/`chunk`/`worker`）：https://www.papaparse.com/docs
- `csv-parse` 官方文档（Node Transform Stream & Async Iterator）：https://csv.js.org/parse/

### 工程选型结论
- **Python 后端**：CSV 使用标准库 `csv`（务必在启动时安全调高 `csv.field_size_limit`）或 `polars.scan_csv`；Excel 统一优先 **`python-calamine`**（兼容 `.xlsx`/`.xls`/`.xlsb`/`.ods` 且速度极快），超大单文件极低内存需求时降级使用 **`openpyxl(..., read_only=True)`**。
- **Node/TS 后端**：CSV 使用 **`csv-parse`**（服务端 `for await` 背压流）+ 浏览器预览用 **`PapaParse`**；Excel 使用 **`ExcelJS` (`WorkbookReader`)**，尽量避开 npm 上的过时 `xlsx@0.18.5`。

---

## 4. 多模型 SDK / 网关抽象对比

Prompt 调优平台需要同时支持 **OpenAI / Anthropic / Google Gemini / 各类 OpenAI-compatible 端点**（如 DeepSeek、vLLM、OneAPI/NewAPI），并可靠采集 **结构化输出**、**Token 用量** 与 **请求耗时** 以写入 **Execution Trace**。

| 评估维度 | Vercel AI SDK (`ai`) | LiteLLM (Python SDK / Proxy) | 官方原生 SDK (`openai` / `anthropic` / `google-genai`) |
| :--- | :--- | :--- | :--- |
| **多模型统一抽象** | **优秀**：统一 `generateText` / `streamText` 接口，通过 `@ai-sdk/openai`、`@ai-sdk/anthropic`、`@ai-sdk/google`、`@ai-sdk/openai-compatible` 切换 | **极佳**：统一 `completion` / `acompletion` 接口覆盖 100+ Provider；通过 `custom_llm_provider` + `api_base` 无缝接入任意 OpenAI-compatible 服务 | **需自建抽象层**：三家接口形状、错误码、流式事件格式完全不同 |
| **结构化输出** | 使用 `generateText` + `Output.object({ schema })` 绑定 **Zod** Schema；`@ai-sdk/openai-compatible` 支持 `providerOptions.openaiCompatible.strictJsonSchema` | 统一透传 `response_format`（`json_object` / `json_schema` `strict: true`），并提供 `supports_response_schema()` 动态探测模型能力 | • **OpenAI**: `responses.parse()` / `beta.chat.completions.parse()` 自动启用 `strict: true` 并暴露 `.refusal`<br>• **Anthropic**: 原生 `output_config: {"format": {"type": "json_schema", "schema": ...}}` 或 `tool_use`<br>• **Gemini**: `GenerateContentConfig(response_mime_type="application/json", response_schema=Pydantic/JSONSchema)` |
| **Token 与成本统计** | 返回统一 `result.usage`（含输入/输出/总 Token 及部分模型的推理/缓存 Token），但不内置实时各家单价计费表 | 返回标准化 `response.usage`（含 `prompt_tokens_details` 缓存详情），并内置 `completion_cost()` / `cost_per_token()` 基于价格表自动算费 | 各家字段名与结构各自独立（如 Gemini 为 `usage_metadata`，Anthropic 为 `input_tokens`/`output_tokens`），需人工归一化 |
| **耗时与观测 (Trace)** | 支持 `telemetry` / `experimental_telemetry` + `@ai-sdk/otel` 导出 OpenTelemetry Span（含 `functionId`、自定义 `metadata` 及隐私脱敏开关 `recordInputs`/`recordOutputs`） | 回调钩子与 Proxy 日志内置 `response_ms` 与 `response_cost`，原生支持对接 Langfuse / OTel / 自定义回调写入 DB | 无内置跨供应商 Trace 框架，需自行记录起止时间戳（TTFT / 总耗时）与原始请求响应 |
| **适用定位** | TypeScript 全栈/Node 服务的最佳模型调用层 | Python 服务（SDK 直调）或跨语言团队独立部署统一模型网关（LiteLLM Proxy） | 仅在需要深度使用某家刚发布独占特性（如特定 Prompt Caching 控制或最新 Beta 参数）时作为底层参考 |

**核心来源**：
- Vercel AI SDK 结构化输出、OpenAI-compatible 与 Telemetry 文档：https://ai-sdk.dev/docs/ai-sdk-core/generating-structured-data 、https://ai-sdk.dev/providers/openai-compatible-providers 、https://ai-sdk.dev/docs/ai-sdk-core/telemetry
- LiteLLM 结构化输出（`response_format`、`supports_response_schema`）与 Token/Cost 文档：https://docs.litellm.ai/docs/completion/json_mode 、https://docs.litellm.ai/docs/completion/token_usage
- OpenAI Structured Outputs（`.parse()`、`.refusal`）：https://platform.openai.com/docs/guides/structured-outputs
- Anthropic Structured Outputs（`output_config.format`）：https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- Google Gemini Structured Outputs（`google-genai`、`response_mime_type`、`response_schema`）：https://ai.google.dev/gemini-api/docs/structured-output

### 工程选型提示
- 并非所有 OpenAI-compatible 网关或开源模型都支持 `strict: true` 的 JSON Schema 约束解码；平台在调用抽象层中必须实现**两级兜底**：「优先开启原生 Structured Outputs / `json_schema` → 不支持时降级为 `json_object` 或 Prompt 约束 + `json-repair`/`jsonrepair` + Pydantic/Zod 校验」。

---

## 5. 异步批处理队列与实时 Execution Trace 推送方案

### 5.1 异步批处理队列选型

面向内部团队“百至数万条”批量 **Workflow** 跑测，任务特点是：**单批次内包含大量独立 Item、高 I/O 并发（受制于各模型 API RPM/TPM 限流）、需持久化进度与支持失败重试/取消**。

| 队列方案 | 运行生态与底层依赖 | 并发模型与 I/O 适配 | 运维复杂度（Docker） | 核心优缺点 |
| :--- | :--- | :--- | :--- | :--- |
| **Taskiq** | Python + Redis / RabbitMQ / NATS 等 | **原生 `asyncio`**，高并发 LLM HTTP 调用开销极低 | 需 Postgres + Redis | 支持类似 FastAPI `Depends` 的依赖注入、Pydantic 友好、社区活跃；适合现代 FastAPI 架构 |
| **ARQ** | Python + 仅 Redis | **原生 `asyncio`**（`max_concurrent_tasks`） | 需 Postgres + Redis | 极简轻量、低延迟；但项目已进入**维护模式**，长期新特性迭代有限 |
| **Celery** | Python + Redis / RabbitMQ | 传统 `multiprocessing` / gevent 池为主 | 较重（Worker + Beat + Flower） | Canvas 复杂 DAG 编排最成熟，但与原生 `asyncio` 混用需桥接，短任务进程开销较高 |
| **BullMQ** | Node.js / TS + Redis | 异步事件循环 + Lua 原子脚本，支持 `concurrency` 配置 | 需 Postgres + Redis | 原生支持 `job.progress()`、`QueueEvents`、Flows (DAG) 与全局 Rate Limiter（非常契合 LLM API 限流） |
| **pg-boss**（或 Postgres `SKIP LOCKED` 队列） | Node.js / TS（或 Python 自研表）+ **仅 PostgreSQL** | `SELECT ... FOR UPDATE SKIP LOCKED` | **极简**：免 Redis，与业务数据同库 ACID 事务 | 百至数万条内部任务完全胜任，无“业务落库成功但入队失败”不一致问题；需注意高频完成记录带来的表膨胀与 Autovacuum 调优 |

**核心来源**：
- Taskiq 官方文档：https://taskiq-python.github.io/
- ARQ 官方文档：https://arq-docs.helpmanual.io/
- Celery 官方文档：https://docs.celeryq.dev/en/stable/
- BullMQ 并发与进度文档：https://docs.bullmq.io/guide/workers/concurrency
- pg-boss（PostgreSQL `SKIP LOCKED`）：https://github.com/timgit/pg-boss

### 5.2 实时进度与 Execution Trace 推送：SSE vs WebSocket

| 对比维度 | Server-Sent Events (SSE / `EventSource`) | WebSocket (`ws` / `wss`) |
| :--- | :--- | :--- |
| **通信方向** | 单向（Server → Client）；暂停/取消/重跑等控制操作通过独立 HTTP POST 发送 | 全双工双向通信 |
| **断线重连与状态恢复** | 浏览器 `EventSource` **原生支持自动重连**，并自动携带 `Last-Event-ID` 请求头实现断线增量续传 | **需手动实现**重连退避算法、心跳与消息序列号补发 |
| **协议与代理兼容性** | 基于标准 HTTP/HTTPS（`text/event-stream`），内网反向代理/Nginx/企业网关兼容性好（需配置关闭代理缓冲 `X-Accel-Buffering: no`） | 需 HTTP Upgrade 握手，部分企业防火墙或未配置 Upgrade 的反代会阻断 |
| **连接数与 HTTP/2** | **HTTP/1.1 下浏览器同域限制 ~6 个并发连接**（多标签页打开多个 SSE 需注意）；**开启 HTTP/2 后可多路复用共享单条 TCP 连接** | 独立 TCP 长连接，不享受 HTTP/2 多路复用 |
| **框架原生支持** | FastAPI 0.135.0+ 原生 `fastapi.sse`（自动输出正确头与 15s 心跳），旧版用 `sse-starlette`；Next.js/Node 用标准 `ReadableStream` | FastAPI / Node `ws` 均支持，但多 Worker 横向扩展时同样需要 Redis Pub/Sub 转发 |

**核心来源**：
- MDN Server-sent events (`EventSource`、`Last-Event-ID`、HTTP/1.1 6 连接限制与 HTTP/2 多路复用)：https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- MDN WebSockets API：https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
- FastAPI 原生 SSE 文档（`fastapi.sse`）与 `sse-starlette`：https://fastapi.tiangolo.com/tutorial/server-sent-events/ 、https://github.com/sysid/sse-starlette

### 实时观测推荐模式
对于 Prompt 调优平台的 **Execution Trace 实时观测**与**批量跑测进度条**：
1. **优先选择 SSE**（而非 WebSocket）：因为前端主要是消费单向增量日志与状态变更（单次运行内可按 Node 输出、Token 流和批次完成数推送），配合 `Last-Event-ID` 可在页面刷新或网络抖动时从持久化 Trace 表/Redis Stream 精确续传。
2. **多容器解耦架构**：`Worker → 写入 PostgreSQL Trace 表 + 发布事件至 Redis Pub/Sub（或基于数据库自增游标轮询/LISTEN-NOTIFY）→ Web API SSE Endpoint 推送给浏览器`，确保浏览器断连不影响后台批跑测继续执行，也便于在内网 Docker Compose 下保证 Nginx/Caddy 开启 HTTP/2 消除 6 连接限制。

---

## 6. 面向 PromptDone 的综合选型建议（供 ADR #6 参考）

1. **首选推荐架构（数据清洗与多模型能力优先）**：**React (TypeScript SPA) + Python FastAPI + Taskiq (Redis Broker) + PostgreSQL**（Docker Compose 部署）。
   - **Dataset 清洗**：`python-calamine`（Excel）+ Python `csv`（调高 `field_size_limit`）/ `polars`（CSV）+ **`json_repair`**（`return_objects=True, ensure_ascii=False`，配合围栏预剥离与 Pydantic Schema 校验）。
   - **多模型与结构化输出**：**LiteLLM SDK**（或独立 LiteLLM Proxy），统一接入 OpenAI / Anthropic / Gemini / OpenAI-compatible 端点，原生采集 `usage`、耗时与估算成本。
   - **批量执行与观测**：**Taskiq** 高并发 `asyncio` Worker 处理百至数万条跑测并支持按 Provider 限流；**FastAPI 原生 SSE (`fastapi.sse`) + Redis Pub/Sub** 推送进度与 **Execution Trace**。
2. **备选全栈 TypeScript 架构（团队纯 TS 且重前后端类型共享）**：**Next.js (Standalone) 或 React + Fastify + BullMQ (Redis) / pg-boss (PostgreSQL)**。
   - **Dataset 清洗**：`ExcelJS` (`WorkbookReader`) + `csv-parse` / `PapaParse` + **`josdejong/jsonrepair`** + Zod 校验。
   - **多模型与结构化输出**：**Vercel AI SDK**（`generateText` + `Output.object()` + `@ai-sdk/openai-compatible` + `@ai-sdk/otel`）。
   - **注意**：即便前端用 Next.js，批量任务仍须交由独立 Node Worker 容器运行，不可依赖 Next.js HTTP 请求生命周期。

---

## 7. 一手权威来源索引

- **JSON 修复与解析**：
  - `mangiucugna/json_repair`: https://github.com/mangiucugna/json_repair
  - `json_repair` LLM 输出示例: https://github.com/mangiucugna/json_repair/blob/main/examples/repair_llm_output.py
  - `josdejong/jsonrepair`: https://github.com/josdejong/jsonrepair
  - `JSON5` 官方规范与仓库: https://json5.org/ , https://github.com/json5/json5
  - `pyjson5` (Python JSON5): https://pypi.org/project/json5/ , https://github.com/dpranke/pyjson5
- **Excel / CSV 流式解析**：
  - `openpyxl` 优化与只读模式: https://openpyxl.readthedocs.io/en/stable/optimized.html
  - `python-calamine`: https://github.com/dimastbk/python-calamine , https://pypi.org/project/python-calamine/
  - `Polars` `read_excel`: https://docs.pola.rs/api/python/stable/reference/api/polars.read_excel.html
  - `Pandas` `read_csv`: https://pandas.pydata.org/docs/reference/api/pandas.read_csv.html
  - Python `csv` 标准库: https://docs.python.org/3/library/csv.html
  - `ExcelJS`: https://github.com/exceljs/exceljs
  - `SheetJS` Node.js 安装文档: https://docs.sheetjs.com/docs/getting-started/installation/nodejs/
  - `PapaParse`: https://www.papaparse.com/docs
  - `csv-parse`: https://csv.js.org/parse/
- **多模型 SDK 与结构化输出**：
  - Vercel AI SDK 结构化输出: https://ai-sdk.dev/docs/ai-sdk-core/generating-structured-data
  - Vercel AI SDK OpenAI-Compatible: https://ai-sdk.dev/providers/openai-compatible-providers
  - Vercel AI SDK Telemetry: https://ai-sdk.dev/docs/ai-sdk-core/telemetry
  - LiteLLM JSON Mode / Structured Outputs: https://docs.litellm.ai/docs/completion/json_mode
  - LiteLLM Token Usage & Cost: https://docs.litellm.ai/docs/completion/token_usage
  - OpenAI Structured Outputs: https://platform.openai.com/docs/guides/structured-outputs
  - Anthropic Structured Outputs: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
  - Google Gemini Structured Output: https://ai.google.dev/gemini-api/docs/structured-output
- **部署、异步队列与实时推送**：
  - Next.js `output: "standalone"`: https://nextjs.org/docs/app/api-reference/config/next-config-js/output
  - FastAPI Server-Sent Events: https://fastapi.tiangolo.com/tutorial/server-sent-events/
  - `sse-starlette`: https://github.com/sysid/sse-starlette
  - Taskiq: https://taskiq-python.github.io/
  - ARQ: https://arq-docs.helpmanual.io/
  - Celery: https://docs.celeryq.dev/en/stable/
  - BullMQ 并发指南: https://docs.bullmq.io/guide/workers/concurrency
  - pg-boss: https://github.com/timgit/pg-boss
  - MDN Server-Sent Events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
  - MDN WebSockets API: https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
