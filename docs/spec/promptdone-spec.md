## Problem Statement

内部 AI 研发与运营团队（含出海本地化团队）在离线与预发布阶段进行 Prompt 工程时，面临以下结构性痛点：

1. **脏数据准备成本高**：业务导出的 Excel、CSV 与 JSON 中频繁混杂带有 Markdown 围栏、双重转义、单引号/尾逗号、Python 字面量或截断的“偏 JSON”文本，现有脚本式预处理缺乏可视化采样预览、修复前后 Diff 留痕、单行人工修补及版本化快照能力。
2. **多节点与多轮调优缺乏系统支撑**：真实业务往往由多个串联或分叉/汇聚的 `Node` 组成 `Workflow`，并涉及历史对话注入或固定多轮追问；团队缺少“修改草稿 → 锁定上游缓存复用 → 对比试跑 → 不可变 `Workflow Run` 快照与双层 `Execution Trace` 追溯”的工程化闭环。
3. **跨版本与多语言评估缺乏可信度量**：单纯靠人工肉眼看几条样本极易引入 `Regression`；多语种出海翻译还面临 `Glossary` 动态命中、`Style Guide` 语气遵循、占位符/ICU 语法防破坏、`do_not_translate` 词条校验以及裁判位置偏差等复杂质量门禁需求。
4. **Bad Case 难以高效流转至外部编码 Harness**：团队在发现系统性失败后，仍需人工复制拼接 Prompt、失败样本、预期输出与运行日志才能交给 Codex 或 Antigravity 等外部编码 Agent 排障或调优，且过程中存在泄露 `Model Provider` API 密钥或超长字段撑爆上下文的风险。

团队需要一个支持**本地 / 内网 Docker Compose 一键部署**、具备**轻量 `Workspace` 隔离**的内部 Prompt 调优与数据清洗工作台 **PromptDone**，打通“数据清洗与版本化 → 多节点 `Workflow` 编排与流式试跑 → 三层质量评估与版本 Diff → AI 辅助调优与 `Context Bundle` 导出”的 P0 核心闭环。

## Solution

构建前后端分离、单仓库（Monorepo）管理的内部 Web 平台 **PromptDone**（React + TypeScript Vite SPA 前端，Python FastAPI + Taskiq 异步 Worker + PostgreSQL + Redis + Nginx 后端基础设施，进程内集成 LiteLLM Python SDK）：

1. **版本化数据清洗管道**：支持 Excel（单 Sheet）、CSV/TSV、JSON/JSONL 导入；首屏 100 条实时采样预览与四角色字段映射（`Input`、`Expected Output`、`Comparison Key`、`Metadata`）；对显式标注的 `JSON` 列执行两级“偏 JSON”修复管道（确定性预处理 + `json-repair`），支持标黄留痕、失败行在线修补/跳过/导出，并发布为不可变 `Dataset Version`（支持基于业务 `Comparison Key` 的追加覆盖，并留存原始上传文件与解析参数）。
2. **独立资产化 `Prompt` 与无环 DAG `Workflow` 工作台**：`Prompt` 拥有可变 Draft 与不可变 `Prompt Version`；`Workflow` 支持无环 DAG 编排、受控 `{{var}}` 与 `History Messages` 插槽、静态历史注入与固定多轮追问、基于 `upstream_subgraph_hash` 的锁定上游缓存复用、4 类试跑模式（单条即时流式、前 N 条/子集、固定 `seed` 随机抽样、全量批量）、并发退避重试与原地断点续跑，并将每次试跑冻结为不可变 `Workflow Run` 与双层 `Execution Trace`。
3. **三层评估、`Prompt Review` 与版本 Diff 报告**：`Eval Suite` 与 `Workflow` 拓扑解耦，每次评分生成不可变 `Evaluation Run`。包含 **Layer 1**（确定性硬门禁，含 `repaired` 严格模式默认判 Fail、占位符/ICU/`do_not_translate` 校验）、**Layer 2**（CoT 先行 + 多维加权归一化 + 异构模型跨家族位置交换成对盲评 + 按需回译辅助）、**Layer 3**（列表内嵌快捷标注与一键入黄金基准集）。配套按需 `Prompt Review`、按 `Comparison Key`（多语言按 `(comparison_key, target_lang)`）对齐且 `Regression` 置顶的双维版本 Diff、结构化/按需 AI Bad Case 聚类，以及任务/裁判/调优三方费用隔离的多口径报告导出。
4. **出海翻译矩阵与本地化资产**：在不设闭盒节点的前提下，为通用 `Node` 提供 `Glossary`（前置词边界/CJK 最长匹配动态子集注入 + Locale 前缀回退）、`Style Guide` 挂载与可选 XLIFF/ICU 前置掩码还原，预置 `Translate-and-Revise` 模板，并支持单次 `Workflow Run` 按 `(Dataset Record × 目标语种)` 矩阵展开。
5. **AI 智能调优与 `Context Bundle` 导出**：支持聚焦单 `Node` 的 `Tuning Suggestion` 半自动调优闭环（正反样本对照、追问微调、锁定版本 Draft 冲突保护、独立成本核算），以及面向 Codex / Antigravity 的 `context-bundle.md` 一键复制与 `.json`/`.zip` 结构化包导出（双场景预设、严禁导出凭证、字段截断与轻量敏感字段掩码）。

## User Stories

### 账号初始化、权限与 `Workspace` 隔离

1. As a 部署管理员, I want through `.env` 预置初始超级管理员账号或通过首次启动引导页创建首个超级管理员, so that 内网部署完成后无需命令行手动插库即可安全进入系统并关闭公开自助注册。
2. As a `Workspace` `admin`, I want 在设置页创建或邀请内部成员并分配 `admin` 或 `member` 角色, so that 团队成员可以协同使用平台资产同时限制敏感密钥配置权限。
3. As a 平台用户, I want 在顶部栏切换所属的 `Workspace`（开箱自带 `Default Workspace`）, so that 不同业务线或项目的 `Dataset`、`Prompt`、`Workflow`、`Eval Suite` 与 `Model Provider` 严格隔离。
4. As a `Workspace` `admin`, I want 在 `Model Providers` 页面配置加密存储的 API Key、Base URL、可选并发/RPM/TPM 限流阈值、自定义单价与结构化输出能力标志并执行连通性探测, so that 团队所有调优与评测任务可以安全、稳定且可计费地调用多厂商大模型。
5. As a 部署管理员, I want 系统首次启动时自动将 `.env` 中的默认 Provider 配置加密写入 `Default Workspace`（仅当无预置记录时），并支持页面手动触发恢复默认配置与离线 CLI 轮换主加密密钥, so that 既能开箱即用又不因重启覆盖团队在页面上修改的配置。

### `Dataset` 导入、偏 JSON 清洗与不可变版本管理

6. As a 数据准备人员, I want 上传 Excel（`.xlsx`/`.xls`）、CSV/TSV 或 JSON/JSONL 文件并配置 Sheet 选择、表头行号、编码/分隔符及嵌套 JSON 路径, so that 各类原始导出的业务评测数据都能被准确解析。
7. As a 数据准备人员, I want 在上传后立即获得前 100 条数据的表头识别、列类型推断（`Text`/`JSON`/`Number`/`Boolean`）与首屏清洗预览, so that 在全量处理前快速验证解析规则是否正确。
8. As a 数据准备人员, I want 将解析后的列可视化映射为 `Input` 变量（支持多列或由单个偏 JSON 列解包展开）、可选 `Expected Output`、可选 `Comparison Key` 与可选 `Metadata`, so that `Dataset Record` 结构能够直接适配下游 `Workflow` 与评估器。
9. As a 数据准备人员, I want 系统仅对显式确认为 `JSON` 类型的列（或需解包的偏 JSON 列）执行“确定性预处理 + `json-repair`”两级修复管道并提供“截断补全”列级开关，同时保留修复前后的 Diff 与 `valid`/`repaired`/`failed` 状态, so that 普通文本中的花括号或代码片段不会被误伤且所有修复动作可审计。
10. As a 数据准备人员, I want 在 ≤500 条时同步全量清洗进入 `Import Draft`，超过 500 条时自动交由后台 Worker 异步流式清洗并实时推送进度, so that 大批量数据导入不会阻塞页面或触发 HTTP 超时。
11. As a 数据准备人员, I want 在 `Import Draft` 中一键筛选置顶 `failed` 失败行、在线单行编辑修复（带实时 JSON 校验）、批量跳过排除或导出错行（含原始列与报错原因）, so that 局部脏数据不会阻断整批数据集的发布。
12. As a 数据准备人员, I want 将校验通过的 `Import Draft` 发布为不可变 `Dataset Version`（`v1`, `v2`, ...），并在版本元信息中保留总行数、直接通过/自动修复/人工修复/跳过统计及原始文件留存, so that 历史跑测结果永远绑定确定快照且支持后续追溯。
13. As a 调优工程师, I want 向已有 `Dataset` 追加新文件生成新的 `Dataset Version`（强制对齐上一版 `Input` 键集合，并在映射了业务 `Comparison Key` 时提示同名冲突覆盖或跳过）, so that 测试集可以安全迭代扩充而不破坏已有 `Workflow` 变量引用。

### `Prompt` 版本管理与多节点 `Workflow` 编排执行

14. As a Prompt 工程师, I want 独立管理可复用的 `Prompt` 资产（维护可变 Draft 与发布不可变编号的 `Prompt Version`），并在其中完整定义 System/User/History 消息模板、默认模型推理参数与输出格式/JSON Schema, so that 同一 `Prompt` 可跨多个 `Workflow` 复用并严格版本溯源。
15. As a Prompt 工程师, I want 在编辑 `Prompt` 时按需选择当前 `Workspace` 的高阶 `Model Provider` 触发 `Prompt Review` 静态体检（可选附带最近 3–5 条代表性 Bad Case），查看按 `High`/`Medium`/`Low` 分级的问题与定位，并在 Diff 视图中一键采纳改写建议到 Draft, so that 在批量跑测前就能系统性消除歧义、格式冲突与冗余 Token。
16. As a Prompt 工程师, I want 在左栏自动折叠为 56px 图标栏的 `Workflow` 详情工作台中，通过 `@xyflow/react` 画布编排无环 DAG 拓扑（支持线性串联、分叉与汇聚），并在顶部面包屑快速切换不同 `Workflow` 与 `Tuning Session`, so that 无需离开工作台即可专注复杂链路编排与多方向实验。
17. As a Prompt 工程师, I want 为每个 `Node` 选择“锁定某个 `Prompt Version`”或“挂载实时 Draft”，并支持在 `Node` 级别临时覆盖模型或推理参数，同时通过 `{{var_name}}`、`{{node_id.output.field}}` 与独立 `History Messages` 插槽映射上游与 `Dataset Record` 变量, so that 既能稳固上游依赖又能灵活对比不同 Prompt 与参数组合。
18. As a Prompt 工程师, I want 在单个 `Node` 内配置静态历史注入（History Replay）或固定多轮追问脚本（Sequential Multi-turn），并在下游引用 `{{node_id.output}}`、`{{node_id.turns}}` 或 `{{node_id.transcript}}`, so that 多轮对话与客服追问类 Prompt 能够在 P0 完整验证。
19. As a Prompt 工程师, I want 当上游 `Node` 已锁定 `Prompt Version` 且在同 `(dataset_version_id, dataset_record_id, target_lang, upstream_subgraph_hash)` 下存在已成功缓存输出时，调试下游 `Node` 默认复用上游缓存（标注 `cache_hit: true`）并支持一键强制重跑上游, so that 调试长链路下游节点时大幅节省模型费用与等待时间。
20. As a Prompt 工程师, I want 使用 4 种试跑模式（单条即时流式调试、前 N 条/指定子集、固定 `seed` 随机抽样、全量批量）触发 `Workflow Run`，并在运行中支持暂停、恢复、取消，或针对未完成/局部失败的 `Workflow Run` 在原地补齐重跑未成功记录, so that 试跑过程可控且不会因偶发限流或超时浪费已完成记录的费用。
21. As a Prompt 工程师, I want 在右侧双层 `Execution Trace` 面板实时查看单条调试的逐节点 Token 流或批量跑测的 Record 级汇总流，点击单条 Record 懒加载按 DAG 拓扑挂载的 `Node` 逐轮 `Turns`、解析状态（`valid`/`repaired`/`schema_failed`）、缓存命中、延迟、Token 与独立冻结费用, so that 任何异常与耗时瓶颈都能精准定位。

### 出海翻译增强与多语言矩阵评估

22. As a 本地化工程师, I want 在 `Workspace` 中维护可复用的 `Glossary`（支持 CSV/XLSX/JSON 导入、`do_not_translate` 标记、`case_sensitive` 开关与 `notes`）和按产品线/语种维护的 `Style Guide`（支持 Locale 前缀如 `es-MX → es` 回退），并一键创建预置的 `Translate-and-Revise` 多节点 `Workflow` 模板, so that 无需从零搭建即可复用标准出海翻译工作流。
23. As a 本地化工程师, I want 在通用 `Node` 上可选挂载 `Glossary`（运行时按前置词边界/CJK 最长匹配动态注入命中的 `{{matched_glossary}}` 子集）与 `Style Guide`，并按需开启 XLIFF 风格占位符掩码与 ICU 片段提取自动还原开关, so that 在保留完整 Prompt 调优自由度的同时保护占位符并控制注入 Token 长度。
24. As a 本地化工程师, I want 在触发 `Workflow Run` 时从单份源文案 `Dataset` 勾选多个目标语种，由引擎按先抽样后展开的方式自动生成 `(Dataset Record × 目标语种)` 矩阵任务并自动注入 `{{target_lang}}` 等变量，在报告中支持汇总与按 `target_lang` 切片分析, so that 一次试跑即可横向对比同一源句在多语种下的译文质量。

### 三层评分体系、版本 Diff 与评估报告

25. As a 质量评估人员, I want 在独立的 `Eval Suites` 页面创建与拓扑解耦的 `Eval Suite`（配置 Layer 1 硬门禁规则、Layer 2 多维加权 Rubric/及格线/位置交换盲评开关/异构裁判模型与按需回译开关），并绑定到 `Workflow` 终端或任意中间 `Node`, so that 同一套评分标准可以跨多个版本稳定复用。
26. As a 质量评估人员, I want 对已完成的 `Workflow Run` 按需触发或修改 Rubric 重跑生成多条不可变 `Evaluation Run`（且同一 `Workflow Run` 下的人工复核标注独立持久化不因重评丢失）, so that 优化评估标准时无需重新花钱重跑业务 `Workflow`。
27. As a 质量评估人员, I want 让 Layer 1 默认开启严格模式（将 `json-repair` 修复后的结果判为 `Fail` 并单列“可修复失败率”，且短路跳过 Layer 2），同时在多语言场景下强制校检占位符/ICU 合法性与 `do_not_translate` 词条, so that 工程侧格式与禁翻风险被零容忍拦截且不浪费裁判费用。
28. As a 质量评估人员, I want 在双版本对比时启用 Layer 2 跨家族异构模型的双向位置交换成对盲评，当正反两次判定相反时自动标记 `Conflict / Tie` 并推入优先人工复核队列, so that 有效消除 LLM-as-a-Judge 的位置偏差与同家族偏好。
29. As a 质量评估人员, I want 在评估结果与版本 Diff 表中内嵌执行 Layer 3 快捷标注（覆写最终 `Pass`/`Fail`/分数、标记内置或自定义错因标签、指定归因 `Node` 与文本批注），并一键将校准样本加入标记为 `is_golden` 的黄金基准 `Dataset` 的 `Import Draft`, so that 人工诊断能够直接沉淀为高可信回归基准集。
30. As a Prompt 工程师, I want 在 `版本对比 Diff & 评估报告` Tab 中对比同 `Dataset` 下的两条 `Workflow Run`（支持提示评估规则不一致并一键统一重评），在同一视图下审阅配置侧 Monaco Prompt/参数 Diff 与按 `Comparison Key`（多语言按 `(comparison_key, target_lang)`）对齐且 `Regression` 置顶的输出侧三方对照 Diff, so that 改版带来的净收益与回退风险一目了然。
31. As a 团队负责人, I want 在评估报告中查看严格隔离的「业务 `Workflow` 费用 / 裁判费用 / 调优辅助费用」、端到端与 `p50`/`p95` 延迟、各维度均分及 `Tuning Session` 跨版本趋势折线图，并导出同 `Workspace` 内只读报告深链、Markdown 汇总报告及明细 CSV/JSON, so that 既能精确核算多方成本又便于内网分享复盘。

### AI 智能调优与 `Context Bundle` 导出

32. As a Prompt 工程师, I want 在评估或 Diff 视图中通过手动勾选或按结构化/AI 语义聚类代表性抽样将 Bad Case（默认上限 15 条，附带集群分布摘要与人工错因批注）送入 `AI 智能调优`, so that 调优模型能获得高信噪比的失败上下文。
33. As a Prompt 工程师, I want 在 `AI 智能调优 & Context Bundle 导出` Tab 中聚焦单个 `Node`，选择 `Model Provider` 并默认附带 2–3 条正向对照样本生成根因归纳与 `Prompt` 改写 Diff，支持多轮追问微调并在采纳时触发锁定 `Prompt Version` 的 Draft 冲突保护弹窗，将完整过程持久化为 `Tuning Suggestion` 并独立核算调优费用, so that 改写过程防过拟合、不误篡改已发布版本且全程可追溯。
34. As a Prompt 工程师或开发工程师, I want 一键复制紧凑型 `context-bundle.md`（带标准任务指引头）或下载结构化 `context-bundle.json` / `.zip` 归档（含 `Prompt 调优预设` 与 `工程排障预设`、5 大区块裁剪预览、2,000 字符单字段截断阈值与轻量敏感字段掩码开关，且绝对剥离任何 API Key 凭证）, so that 可以安全高效地将诊断上下文交付 Codex 或 Antigravity 继续深入解决。

## Implementation Decisions

### 1. 总体架构与核心模块划分（遵循 ADR-0001）

- **前后端分离单仓库（Monorepo）**：
  - **前端模块**：React + TypeScript（Vite SPA）、Tailwind CSS + shadcn/ui（Radix 原语）、`@xyflow/react`（无环 DAG 画布）、TanStack Table / Virtual（大表虚拟滚动）、TanStack Query（服务端状态）+ Zustand（画布与本地草稿状态）、Monaco Editor（含 DiffEditor）。通过 FastAPI 导出的 OpenAPI Schema 使用 `openapi-typescript` + `openapi-fetch` 自动生成强类型 API 客户端并纳入构建校验。
  - **后端模块**：Python FastAPI（异步 Web 与 SSE 端点）+ Taskiq 异步 Worker（`taskiq-redis` + `taskiq-fastapi`），共享同一套 Python Domain/Model 包；数据层使用 SQLAlchemy 2.0 (async) + `asyncpg` + Alembic + Pydantic v2 + PostgreSQL（核心动态数据与快照使用 `JSONB`）+ Redis；文件读写抽象统一 `StorageBackend` 接口，P0 默认写入 Docker Local Volume，预留 S3/MinIO 扩展点。
  - **多模型调用适配模块**：进程内直接集成 LiteLLM Python SDK（`litellm.acompletion`），不额外部署独立 LiteLLM Proxy 容器；封装统一模型调用网关，承担结构化输出三级兜底（`strict` JSON Schema → `json_object` → `json-repair` + Pydantic 校验）、Redis 分布式限流与退避重试、自定义单价/能力覆盖与费用冻结。
  - **内网 Docker Compose 网关与入口**：Nginx 前置统一入口，托管前端静态资源并反代后端 API；开启 HTTP/2，且对 SSE 路径强制关闭缓冲（`proxy_buffering off` 与响应头 `X-Accel-Buffering: no`）以消除浏览器并发连接限制并保障流式实时性。

### 2. 轻量账号体系、`Workspace` 隔离与密钥生命周期

- **账号引导与注册边界**：
  - 平台关闭公开自助注册。支持在 `.env` 配置初始超级管理员凭证（由启动脚本幂等 Seed）；若 `.env` 未配置且 `users` 表为空，首次访问 Web 自动进入**一次性初始化管理员引导页**，提交后关闭初始化入口。
  - `users`（含 `is_superadmin`）、`workspaces`（启动自动初始化 `Default Workspace`）与 `workspace_members`（`admin` / `member` 两级角色）实现内部团队隔离。`admin` 负责成员创建/邀请与 `Model Provider` 管理；`member` 负责资产编辑、跑测与标注。
- **密钥安全与轮换**：
  - API 与 Worker 启动时强校验合法 `APP_ENCRYPTION_KEY`（`Fernet` 对称加密密钥），缺失或非法直接拒绝启动。
  - 首次初始化 `Default Workspace` 且无 `source = "env_seed"` 记录时，将 `.env` 预置的默认 `Model Provider` 加密入库；此后以 DB 为准，页面仅提供显式「恢复 `.env` 默认配置」动作。
  - 所有接口返回密钥仅保留前后掩码（如 `sk-****1234`），编辑留空保持原密文；严禁任何接口与 `Context Bundle` 导出明文密钥。提供离线 CLI 命令支持主加密密钥轮换重加密。
- **限流与单价能力回退**：
  - Redis 按 `model_provider_id`（支持 `model_name` 覆盖）维护可选三限流阈值：`max_concurrency`（带 TTL 租约信号量）、`rpm_limit`（滑动窗口）、`tpm_limit`（调用前按渲染消息估算试扣，返回 `usage` 后按实校准差额）。限流排队或上游返回 `429` / 瞬时 `5xx` 时按 `Retry-After` 优先执行指数退避 + 随机 Jitter（默认最多重试 3 次，可在 `Workflow Run` 覆盖）。
  - 单价与结构化能力优先读取平台自定义覆盖配置，缺失项回退 LiteLLM 内置 `model_cost` 与能力字典；未知模型单价记 `0` 并标记 `cost_estimated: false`。每次执行的 `cost_usd` 当场随记录持久化冻结，不随日后改价回溯变动。
  - 提供 `Model Provider` 连通性与能力探测端点，自动校验密钥/Base URL 并可回填 `strict` JSON Schema / `json_object` 支持标志。

### 3. 数据清洗管道与不可变 `Dataset` 模型

- **多格式解析边界**：
  - Excel（`.xlsx`/`.xls`，基于 `python-calamine`）：P0 默认读取首个非空 Sheet 并支持在多 Sheet 时下拉选定 1 个目标 Sheet；默认第 1 行为表头（允许调整行号），合并单元格按向下/向右填充展开。
  - CSV/TSV：自动探测编码（UTF-8 / UTF-8-SIG / GBK / GB18030）与分隔符（`,`、`\t`、`;`）并允许手动覆盖；调高字段大小读取上限以兼容长 Prompt 上下文。
  - JSON/JSONL：JSONL 逐行解析；标准 JSON 顶层为数组时直接展开，嵌套对象时支持点分路径 / JSONPath 提取目标数组。
- **四角色映射与显式列级两级偏 JSON 修复**：
  - 每条 `Dataset Record` 映射为 `Input`（必选 1..N 字段，支持多列或单个偏 JSON 列解包展开）、`Expected Output`（可选 0..1）、`Comparison Key`（可选 0..1）、`Metadata`（可选 0..N）。
  - 仅对用户显式确认为 `JSON` 类型的列（或用于解包的列）执行两级偏 JSON 修复：
    1. **确定性预处理**：去除 BOM/零宽控制字符、剥离 Markdown 代码块围栏及前后杂文、递归解包多重转义的字符串化 JSON；
    2. **启发式修复**：调用 `json-repair` 修复单引号、尾逗号、未转义引号换行与 Python 字面量（`True`/`False`/`None`）；缺失闭合括号“截断补全”提供列级开关（默认开启并标黄，可切为标红失败）。
  - 每字段持久化 `valid` / `repaired` / `failed` 状态与原始文本快照。
- **采样/异步边界、兜底与版本演化**：
  - 上传后流式解析前 **100 条**实时返回映射与清洗预览；确认后 ≤500 条同步写入 `Import Draft`，>500 条交由 Taskiq Worker 异步流式入库并推送 SSE 进度。
  - `Import Draft` 内支持失败行筛选置顶、在线单行编辑（含实时 JSON 校验）、批量跳过、导出错行 CSV/JSONL；当保留行均合法时可发布为不可变 `Dataset Version`（`v1`, `v2`, ...），关联持久化原始上传文件（默认上限 100MB，可配置）及解析参数。
  - 全量物化每个 `Dataset Version` 的 `dataset_records`（`import_drafts` / `import_draft_records` 与 `dataset_versions` / `dataset_records` 分表存储）。若未显式映射业务 `Comparison Key`，系统生成单版本内唯一的内部键；仅当映射了业务 `Comparison Key` 时，Append 新版本触发跨版本同名冲突提示（「新行覆盖（默认）」或「跳过重复行」），且 Append 必须完成对上一版 `Input` 键集合的映射。

### 4. `Prompt`、`Workflow` DAG、多轮对话、缓存复用与 `Execution Trace`

- **资产与绑定语义**：
  - `Prompt` 为独立资产，含可变 Draft 与不可变 `Prompt Version`（冻结消息模板、默认模型推理参数、输出格式/JSON Schema）。`Node` 可锁定指定 `Prompt Version` 或挂载实时 Draft，并支持节点级参数临时覆盖。
- **DAG 拓扑、变量与多轮语义**：
  - P0 支持标准无环 DAG（线性、分叉、汇聚）。模板使用受控 `{{var_name}}` 与上游引用 `{{node_id.output}}` / `{{node_id.output.field}}`；内置独立 `History Messages` 插槽（置于 System 与当前 User 之间），直接绑定消息数组变量。
  - 节点配置 `JSON` / `JSON Schema` 输出时先尝试原生解析，失败后自动降级 `json-repair` 修复（标记 `repaired`），仍不满足 Schema 则标为 `schema_failed` 并阻断下游字段消费（下游依赖节点自动记为 `skipped (upstream_failed)`）。
  - 单个 `Node` 支持**静态历史注入（History Replay）**与**固定多轮追问脚本（Sequential Multi-turn）**，多轮节点对外默认 `{{node_id.output}}` 指向最后一轮助手回复，同时暴露 `{{node_id.turns}}` 与 `{{node_id.transcript}}`。
- **上游缓存复用与 4 类试跑模式**：
  - 当上游 `Node` 锁定 `Prompt Version` 且同 `(dataset_version_id, dataset_record_id, target_lang, upstream_subgraph_hash)` 存在成功记录时，下游调试默认复用缓存输出（标记 `cache_hit: true` 及来源执行 ID，支持一键强制重跑上游）。
  - 试跑模式全覆盖：① 单条即时流式（不进 Taskiq 队列，FastAPI 异步直连引擎推流 SSE 并落库为单条 `Workflow Run`）；② 前 N 条 / 指定子集；③ 固定 `seed` 随机抽样；④ 全量批量。
  - 每次试跑在所属 `Tuning Session` 下创建不可变 `Workflow Run` 快照，冻结拓扑、各节点实际 Prompt 文本/参数、绑定的 `Dataset Version`、抽样 `seed` 及翻译资产内容。因暂停或局部失败未完成的 `Workflow Run` 支持原地断点续跑/仅重跑失败项。
- **双层 `Execution Trace` 与 PostgreSQL 核心表边界**：
  - 全核心业务实体主键统一采用 **UUID**；顶层可变资产（`datasets`、`prompts`、`workflows`、`eval_suites`、`glossaries`、`style_guides`、`model_providers`）统一使用 `archived_at` 软删除。
  - `Execution Trace` 拆分为 **`execution_traces`**（Record 级总览与 `worker_lease_until` 租约）与 **`node_executions`**（`Node` 级明细、含 `turns` `JSONB`、缓存哈希与复合索引 `(dataset_version_id, dataset_record_id, target_lang, upstream_subgraph_hash)`）。列表页分页加载 `execution_traces`，点开详情懒加载 `node_executions`。
  - Layer 3 人工复核标注存储在按 `execution_trace_id` 唯一索引的独立 1:1 表 **`trace_annotations`** 中，确保针对同一 `Workflow Run` 多次切换 Rubric 重跑 `Evaluation Run` 时人工标注永不丢失。
  - 混合快照策略：不可变资产（`Dataset Version`、已发布 `Prompt Version`）使用外键引用；触发瞬间可变配置（DAG 拓扑、Draft 文本、参数、`Glossary`/`Style Guide`、裁判 Rubric 与模型）冻结为 `workflow_runs.snapshot_json` 与 `evaluation_runs.snapshot_json`。

### 5. Taskiq 状态机与双协议 SSE 事件契约

- **微批调度与租约自愈**：
  - 批量任务（>500 行清洗、批量/抽样 `Workflow Run`、批量 `Evaluation Run`、按需 AI 聚类）走 Taskiq Worker；批量 `Workflow Run` 采用「Run 主控状态 + 分片微批领取（如每片 50 条）+ 进程内 `asyncio.Semaphore` 并发控制 + Redis 控制标志 `run:{id}:control`（`running`/`pausing`/`cancelling`）」。
  - Run 状态机：`pending` → `running` → `pausing` → `paused`（可恢复 `running`）、`cancelling` → `cancelled`，终态 `completed` / `partial` / `failed`。单条 `Execution Trace` 状态：`pending` → `running` → `succeeded` / `failed` / `partial` / `cancelled` / `skipped`。
  - 所有状态跃迁在 PostgreSQL 内通过条件原子更新（`UPDATE ... WHERE status IN (...)`）防重；执行中记录携带 `worker_lease_until`（租期 60s，Worker 每 15s 续租），超时未续租自动回收为 `pending`。
- **Redis Stream SSE 总线与 `Last-Event-ID` 续传**：
  - 批量任务使用 Redis Stream（`sse:run:{run_id}`，带 `MAXLEN` 与短 TTL）+ DB 终态兜底。客户端重连携带 `Last-Event-ID` 时先查 DB 是否已终态（已终态直接下发终态汇总并关闭），否则从 Stream 补发后阻塞推送，每 15s 发送心跳。
  - **批量任务事件**：`run.started`、`run.status_changed`、`trace.batch_completed`（按 **250ms 或满 10 条** 节流聚合推送）、`run.completed` / `run.failed`。
  - **单条调试事件**：`node.started`、`node.token`、`node.cache_hit`、`node.completed` / `node.failed`、`run.completed` / `run.failed`。

### 6. 出海翻译增强与多语言矩阵语义

- **通用 `Node` 挂载与预置模板**：
  - 提供一键新建 `Translate-and-Revise` 多节点 `Workflow` 模板。通用 `Node` 可挂载 `Workspace` 级 `Glossary`（CSV/XLSX/JSON 导入，含 `source_term`、目标语种译文、`do_not_translate`、`case_sensitive`、`notes`）与 `Style Guide`（按产品线/语种定义，支持 Locale 前缀回退）。
  - 注入采用前置词边界（拉丁语系）与 CJK 子串最长优先动态子集匹配，仅将命中词条注入 `{{matched_glossary}}`。
  - `Node` 提供默认关闭的「XLIFF 风格占位符掩码 `<ph id="0"/>`」与「ICU MessageFormat 片段提取」自动前置掩码与后置还原开关。
- **矩阵展开与复合对齐**：
  - 单次 `Workflow Run` 支持勾选多个目标语种，按先抽样源 `Dataset Record` 再展开选中语种的规则生成 `(Dataset Record × 目标语种)` 矩阵任务，自动注入 `{{target_lang}}`（及可选 `{{source_lang}}`、`{{country}}`）；若 `Dataset Record` 自身已含 `target_lang` 列则按行直跑不二次展开。
  - `Execution Trace` 与版本 Diff 使用 `(comparison_key, target_lang)` 复合对齐键，支持按语种切片聚合。

### 7. 三层评分、`Prompt Review`、版本 Diff 与报告导出

- **Layer 1 / Layer 2 / Layer 3 评估规则**：
  - **Layer 1（确定性硬门禁）**：校验 JSON Schema、正则/关键词、长度/耗时/费用阈值、与 `Expected Output` 精确比对、多语言占位符/ICU 语法与 `do_not_translate` 原样保留（普通双语术语默认交 Layer 2 裁决，支持词条级 `strict_match` 开关）。JSON Schema 校验默认开启严格模式（`repaired` 判 Fail，报告单列“可修复失败率”），支持手动放行。Layer 1 失败默认短路跳过 Layer 2（可配置强制全评）。单条即时调试默认仅自动跑 Layer 1，需手动点击“试评单条”才触发 Layer 2。
  - **Layer 2（LLM-as-a-Judge）**：支持多维 Rubric（二元 Pass/Fail 或 1–5 分制，支持 MQM 映射），加权归一化为 0–100 分（默认及格线 80 分，可调）。标配「先 CoT 评判理由，再输出结构化分数」；支持单跑参考 `Expected Output`、对比模式开启异构跨家族模型**双向位置交换成对盲评**（正反结果相反自动标为 `Conflict / Tie` 并推入人工复核队列），以及按需开启**异构模型回译语义一致性辅助校验**（持久化至 `evaluation_results`）。
  - **Layer 3（人工复核与黄金集）**：列表内嵌快捷覆写判定、错因标签、归因 `Node` 与批注，并支持一键将校准后样本加入 `is_golden = true` 的黄金 `Dataset` 的 `Import Draft`。
- **`Prompt Review`、版本 Diff 与 Bad Case 聚类**：
  - `Prompt` 编辑器内按需调用自选高阶 `Model Provider` 对 Draft 或版本执行静态 `Prompt Review`（内置多维审查模板 + 自定义 Rubric，可选附带 3–5 条 Bad Case），结果存入 `prompt_reviews`（独立记账），输出按 `High`/`Medium`/`Low` 分级的发现与可一键采纳的 Diff 改写。
  - 同 `Dataset` 下任意两条 `Workflow Run` 支持版本 Diff（配置不一致时提示一键统一重评；版本/范围不一致时按 `Comparison Key` 并集对齐并高亮 `Only in Baseline/Candidate`，支持切换仅看交集）。状态迁移矩阵包含 `Regression` / `Improvement` / `Both Fail` / `Both Pass` / `Missing or Error`，默认 `Regression` 置顶并支持词级三方对照。
  - Bad Case 支持基于 Layer 1 规则/Layer 2 最低分维/人工标签的**结构化分组聚类**，以及按需点击触发的 **AI 语义归因聚类**。
- **报告口径与导出**：
  - 严格分离业务 `Workflow` 费用、裁判费用与调优辅助费用；提供 `p50`/`p95` 延迟、通过率/可修复失败率、及格率及 `Tuning Session` 跨 Run 趋势折线图。
  - P0 导出形式：登录态校验下的**同 `Workspace` 只读报告深链**（指向不可变 `Workflow Run` / `Evaluation Run` 视图，禁止编辑与重跑）、**Markdown 汇总报告**及**明细 CSV / JSON**。

### 8. AI 智能调优与 `Context Bundle` 导出规格

- **单 `Node` 聚焦调优闭环（`Tuning Suggestion`）**：
  - 每次调优仅聚焦单个目标 `Node`，将上游输出与拓扑作为只读上下文，严禁 AI 自动修改 `Eval Suite` 或直接发布 `Prompt Version`。
  - 支持手动勾选或按聚类代表性抽样入篮（默认上限 15 条，头部保留全量与聚类分布摘要），默认附带 2–3 条正向对照样本防过拟合；生成根因归纳与 `Prompt` 改写 Diff（可选参数建议），支持多轮追问微调。
  - 若目标 `Node` 锁定了不可变 `Prompt Version`，采纳时弹窗引导写入对应 `Prompt` Draft（若 Draft 有未发布改动则先展示 Diff 确认合并/覆盖）并切换该 `Node` 为挂载 Draft，或另存为新 `Prompt`。全过程记录持久化至 `tuning_suggestions`（独立记账 `cost_usd`）。
- **`Context Bundle` 导出与脱敏**：
  - 提供两种互补形态：① 一键复制 / 下载单文件 `context-bundle.md`（带标准任务指引头）；② 结构化包 `context-bundle.json` / `.zip`（拆分 `summary.md`、`prompt`、`schema.json`、`bad_cases.jsonl` 等）。
  - 包含 5 大区块（目标资产与配置、命中的领域上下文、评估标准摘要、全局失败画像、精选 Bad Case 明细），提供 **Prompt 调优预设（默认）** 与 **工程排障预设**（额外含 DAG JSON、变量映射、`json-repair` 前后对比、完整错误栈与排障指引语）及弹窗裁剪预览。
  - 强制剥离一切凭证；P0 提供单字段截断阈值（Markdown 默认 2,000 字符标注 `[truncated]`，`.zip` 可勾选全量）及按字段名/简单正则的轻量敏感字段掩码开关。

### 9. 信息架构与交互骨架（来自原型 `prototype/ui-ia` Variant A / `A1`）

- **全局导航结构**：顶部提供 `Workspace` 切换器与用户菜单；左侧主导航分三组：① 调优核心（`Workflows`）；② Workspace 资产（`Datasets`、`Prompts`、`Eval Suites`、`Glossary & Style Guide`）；③ Workspace 设置（`Model Providers`、Workspace 成员管理）。
- **List-Detail 与自适应折叠（`A1` 决策）**：点击主导航项先进入全宽目录索引页（支持搜索、标签筛选、从模板新建或导入）；进入具体 `Workflow` 详情工作台后，左侧主导航自动折叠为 56px 图标栏（可手动展开固定），顶部面包屑栏内置 `Workflow` 快速切换下拉与 `Tuning Session` 切换/新建下拉。
- **`Workflow` 工作台四大子 Tab**（按原型定稿结构组织）：
  1. `编排与调优 Studio`（顶部试跑控制条 + 左 DAG 画布/节点挂载 + 中 Monaco Prompt 编辑与 `Prompt Review` + 右实时 SSE `Execution Trace` 与单条 L1 门禁）
  2. `Workflow Runs`（不可变运行快照列表、三方隔离费用展示与勾选发起对比）
  3. `版本对比 Diff & 评估报告`（对齐校验与重评入口、指标卡、趋势图、结构化/AI 聚类、Regression 置顶 Diff 表与 L3 快捷标注/入黄金集/入篮、报告导出）
  4. `AI 智能调优 & Context Bundle 导出`（单 Node 聚焦 `Tuning Suggestion` 闭环与 `Context Bundle` 双预设导出弹窗）

## Testing Decisions

采用“最高层外部行为接缝、最小化 Mock”的验证策略：

1. **后端最高层行为接缝（FastAPI HTTP / SSE 端点 + Taskiq 执行链路）**：
   - 使用真实测试 PostgreSQL（执行 Alembic 迁移）与真实 Redis 实例运行端到端接口与 Worker 集成测试（Taskiq 测试模式或独立 Worker 进程），仅在最外层模型调用边界注入一个可编程的 **Fake/Stub `litellm.acompletion` 适配器**（模拟正常 JSON、畸形偏 JSON、流式 Token 分片、429 限流 `Retry-After`、位置偏差响应等）。
   - 核心覆盖的外部行为（不测内部私有辅助函数）：
     - 账号初始化（`.env` Seed / 首启引导关闭自注）与跨 `Workspace` 越权隔离；
     - Excel/CSV/JSON 导入 100 条预览、两级 `json-repair` 状态保留、失败行修补/跳过/导出、不可变 `Dataset Version` 发布与基于 `Comparison Key` 的 Append 冲突策略；
     - 无环 DAG 变量注入、两级解析降级、`upstream_subgraph_hash` 缓存命中与强制重跑、4 种试跑模式、原地断点续跑、条件原子状态跃迁与 `worker_lease_until` 租约超时回收；
     - 单条直连 SSE 与批量 Redis Stream `Last-Event-ID` 断线续传及 250ms/10 条节流协议；
     - Layer 1 严格模式默认判 Fail、Layer 2 位置交换冲突推入人工、Layer 3 标注跨 `Evaluation Run` 重评不丢失、费用三方隔离与单价回退；
     - `Glossary` 词边界/CJK 注入、Locale 前缀回退、XLIFF/ICU 掩码还原与 `(comparison_key, target_lang)` 矩阵对齐；
     - `Tuning Suggestion` 锁定版本 Draft 冲突保护，以及 `Context Bundle` 凭证零泄露、截断与轻量掩码校验。
2. **前端交互接缝（Vitest + RTL + MSW & Playwright E2E）**：
   - 组件与页面状态流使用 Vitest + React Testing Library 配合 MSW（对齐 `openapi-typescript` 生成的契约与 SSE 事件流），验证 `Import Draft` 映射修复、`Studio` DAG 与折叠导航切换、`Diff` 标注与 `Context Bundle` 裁剪预览。
   - 使用 Playwright 对 Docker Compose（Nginx + SPA + FastAPI + Worker + PostgreSQL + Redis + Stub 模型端点）跑通核心全链路端到端冒烟测试。

## Out of Scope

### 明确的永久非目标（来自地图边界）

- 面向外部商业客户的 SaaS 多租户计费与复杂细粒度 RBAC 权限体系（仅做内部团队基础 `Workspace` 与 `admin`/`member` 区分）。
- 分布式大数据 ETL 引擎（如 Spark/Flink，定位百至数万条/批）。
- 线上生产流量的实时 API 网关与灰度路由（定位离线/预发布调优，产物供导出使用）。
- 平台内直接托管或自动调度外部 Agent（Codex / Antigravity）修改业务代码仓库（仅提供标准化 `Context Bundle` 复制/导出）。

### P1 / P2 后续路线图（不纳入本 P0 Spec 拆票范围）

- **数据层 P1/P2**：合并同表头多 Sheet Excel、基于已存原文件一键重洗（Re-clean）UI 全流程强化、自定义 JSON Schema 结构校验导入、版本间 `Dataset Record` 级 Diff 视图、失败行批量查找替换/规则重写、外部数据源/数据库直连定时同步、AI 自动字段映射与样本合成扩增。
- **Workflow 层 P1**：条件分支路由（Conditional Routing）与自动循环重试（Loop）、动态 AI 用户模拟器（User Simulator）、按 `Metadata` 分层抽样、独立的整体 `Workflow Version` 发布机制。
- **评估与报告 P1**：独立盲审分发队列与 Cohen's Kappa ($\kappa$) 一致性校准看板、`Metadata` 多维交叉分析热力图、PDF 报告导出、免登录匿名签名外链分享 Token。
- **翻译层 P1/P2**：TBX/TMX 格式导入、语义向量召回术语、本地部署神经无参考 QE 模型（CometKiwi / xCOMET / MetricX 外部服务端点接入）。
- **调优与安全 P1/P2**：无人值守全自动多轮自迭代（Auto-Tuning Loop）、基于 NLP/NER 的自动全库敏感实体 PII 识别。

## Further Notes

- **地图与决策溯源**：本 Spec 综合了规划地图 [#1](https://github.com/AlexdanerZe/PromptDone/issues/1) 及其全部子决策 Ticket（#2–#10、#12）及收口审核 Ticket（[#11](https://github.com/AlexdanerZe/PromptDone/issues/11)）。
- **规范词汇表与架构决策**：开发与拆票时须严格遵循仓库根目录 `GLOSSARY.md` 中的权威术语定义及 `docs/adr/0001-tech-stack-and-deployment-architecture.md` 的架构约束。
- **UI 信息架构原型参考**：交互布局与组件组织可参考仓库 `prototype/ui-ia` 分支中的 `prototype/ia-prototype.html`（Variant A / `A1`）。
- **技术与领域调研报告**：技术栈与畸形 JSON 修复调研见 `research/tech-stack-and-json-repair` 分支；出海翻译与 LLM-as-a-Judge 调研见 `translation-and-llm-eval` 分支。
