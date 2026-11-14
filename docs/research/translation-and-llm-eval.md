# 出海翻译工作流与 LLM-as-a-Judge 评估报告实践调研

> **关联 Ticket**: [AlexdanerZe/PromptDone#5](https://github.com/AlexdanerZe/PromptDone/issues/5)  
> **调研范围**: (1) 出海翻译场景在 LLM 工作流中的最佳实践；(2) 成熟开源 Prompt 评估与可观测工具的评估对比机制与三层评分设计；(3) Prompt 版本 Diff、Output Diff 与汇总报告的业界设计模式与关键指标。  
> **术语约定**: 本文第 4 节将调研结论映射到 [PromptDone 领域术语表](../../GLOSSARY.md)（**Dataset**、**Node**、**Workflow**、**Execution Trace**、**Tuning Session**、**Context Bundle**）。

---

## 1. 出海翻译场景在 LLM 工作流中的最佳实践

### 1.1 术语表（Glossary）约束注入

在软件本地化与专业领域翻译中，直接使用通用 LLM 翻译容易出现“术语漂移”（同一概念在不同句子/语种中被自由发挥）或忽略品牌词/专有名词的问题。业界与学术界主要有三类约束模式：

1. **传统 NMT 原生硬约束 API（作为对比基线或混合路由层）**
   - **DeepL Glossaries API**：通过 `/v2/glossaries` 或 `/v3/glossaries` 上传 TSV/CSV 双语术语对，并在翻译请求中传入 `glossary_id`。其优势在于支持目标语言的**形态屈折变化（morphological inflection）**，能根据目标语语法自动适配词形（格、性、数、时态）而非生硬字符串替换，但支持的语言对相对有限。  
     来源：[DeepL API Reference — Create a glossary](https://developers.deepl.com/api-reference/glossaries/create-a-glossary)
   - **Google Cloud Translation Advanced (v3) Glossaries**：将 CSV/TSV/TMX 格式术语库托管在 Cloud Storage 并通过资源 ID 绑定翻译请求，支持单向/等效术语集（equivalent term sets），覆盖 100+ 语种。  
     来源：[Google Cloud Translation — Creating and using glossaries](https://docs.cloud.google.com/translate/docs/advanced/glossary)

2. **LLM 上下文注入：动态筛选 + 结构化约束**
   - **按片段动态检索注入（Dynamic Term Matching / RAG）**：当全量术语表包含数百至数千条词条时，直接全部塞入 System Prompt 会浪费大量 Token 并分散模型注意力；常见做法是在预处理阶段通过大小写无关匹配、词干匹配或子串/向量检索，仅提取**当前待翻译文本中命中的术语子集**注入 Prompt（如 `{"Workspace": "工作区", "Execution Trace": "执行追踪"}`），并附带 `do_not_translate`（品牌名/代码标识不翻译清单）与词性/语境说明。
   - **区域/文化变体参数化**：例如吴恩达开源的 `andrewyng/translation-agent` 在核心 `translate(source_lang, target_lang, source_text, country)` 接口中显式引入 `country` 参数，将术语与习惯用语精确约束到区域变体（如面向墨西哥的西班牙语或面向巴西的葡萄牙语），避免跨区用词偏差。  
     来源：[GitHub — andrewyng/translation-agent](https://github.com/andrewyng/translation-agent)

3. **Translate-and-Revise / Reflection 多步约束修正**
   - **学术依据**：论文 *Translate-and-Revise: Boosting Large Language Models for Constrained Translation*（Huang et al., 2024）指出，在长上下文或复杂句式下，单轮 LLM 生成容易遗漏 Prompt 中的词汇约束；采用“初译（Translate）→ 约束核查与修订（Revise）”的两阶段/迭代框架，可显著提升术语命中率并保持译文流畅度。  
     来源：[arXiv:2407.13164 — Translate-and-Revise](https://arxiv.org/abs/2407.13164)
   - **开源工作流实现**：`andrewyng/translation-agent` 将翻译拆分为 **Initial Translation → Reflection（语言专家视角审校误译、遗漏、语气与本地化自然度）→ Improvement（结合反思意见重写）** 三步链路；在工程上非常适合将“术语表校验结果”和“占位符检查报错”作为 Reflection 阶段的输入，驱动自动修复。  
     来源：[GitHub — andrewyng/translation-agent](https://github.com/andrewyng/translation-agent)

---

### 1.2 占位符、富文本标签与 ICU MessageFormat 保护

LLM 倾向于将输入视为自然语言，直接送入原始 UI 文案常导致 `{userName}` 被翻译成 `{用户名}`、`%s` 丢失或 `{count, plural, ...}` 花括号与关键字被破坏。工程界普遍采用**“前置掩码/AST 拆解 + 提示词约束 + 后置确定性校验与自愈”**三道防线：

1. **XLIFF 风格内联掩码（Inline Tag Masking）**
   - 参考 OASIS XLIFF 2.0 标准中的内联元素规范（`<ph>` 独立占位符、`<g>` 成对包裹标签、`<x/>` 空占位符），在调用 LLM 前用正则或词法扫描器将变量（`{{var}}`、`{0}`、`%s`、`${count}`）与 HTML/Markdown 标签替换为与语言无关的不可变标记（如 `<ph id="1"/>` 或 `__PH_1__`），待 LLM 输出后再还原，并提示模型严格保留标记位置与属性。  
     来源：[OASIS XLIFF Version 2.0 Specification](https://docs.oasis-open.org/xliff/xliff-core/v2.0/xliff-core-v2.0.html)

2. **ICU MessageFormat AST 结构拆解**
   - 对于含复数（`plural`）、选择（`select`）及嵌套花括号的 ICU 字符串，直接交给 LLM 极易译坏关键字（如把 `plural` 或 `other` 翻译掉）或遗漏目标语言的 CLDR 复数分支。
   - **最佳实践**：使用 `@formatjs/icu-messageformat-parser` 等 AST 解析器将 ICU 字符串解析为抽象语法树，**锁定外层语法骨架与变量节点**，仅抽取可翻译文本叶子节点与富文本标签上下文送入 LLM；同时在目标语言生成分支时按 CLDR 规则补全所需复数分类（如英语 2 类、俄语/阿拉伯语多类），并在输出后重新用 ICU Parser 做语法校验。  
     来源：[FormatJS — Intl MessageFormat Docs](https://formatjs.github.io/docs/intl-messageformat/)

3. **后置确定性校验与自动修复（Self-Healing Retry）**
   - 译文返回后立即执行零成本代码校验：
     - **集合等价性校验（Placeholder Parity）**：抽取原文与译文的全部占位符集合，校验数量、变量名与顺序/配对标签是否完全一致；
     - **语法解析校验**：针对目标格式运行 ICU Parser、JSON Schema 或 XML 解析；
     - **局部重试**：仅对校验失败的条目构造包含精确错误原因的修正 Prompt 重试，避免整批重跑。

---

### 1.3 多语种批量并发、长文本分块与回译/语义一致性校验

1. **多语种并发与分块（Chunking）工程模式**
   - **Token 分块与增量 Diff**：开源 CLI 工具 `lobehub/lobe-i18n`（`@lobehub/i18n-cli`）的实践具有代表性：通过 `splitToken` 按 Token 阈值自动切分大型 JSON/Markdown 本地化文件以防长上下文遗漏键；默认 `temperature: 0` 保证翻译稳定性；通过 `concurrency`（默认 5）控制跨文件与跨目标语种的并发队列，配合指数退避（Exponential Backoff）规避 429 限流；并基于键 Diff 实现**增量翻译**，仅翻译新增或变更键。  
     来源：[GitHub — lobehub/lobe-i18n](https://github.com/lobehub/lobe-i18n)
   - **长文档上下文分块**：`andrewyng/translation-agent` 提供了 `calculate_chunk_size` 与 `multichunk_*` 工具链，将超长文本按语义边界拆块后分块执行“初译 → 反思 → 修订”，保证长文本不截断且前后连贯。  
     来源：[GitHub — andrewyng/translation-agent](https://github.com/andrewyng/translation-agent)
   - **异步 Batch API 降本（适合离线批量出海）**：
     - **OpenAI Batch API**：以 `.jsonl` 提交请求（单批最多 50,000 请求 / 200 MB），独立限流池、24 小时异步 SLA、**Input/Output 均享 50% 价格折扣**。  
       来源：[OpenAI API Docs — Batch API](https://developers.openai.com/api/docs/guides/batch)
     - **Anthropic Message Batches API**：原生 `requests[]` 批量提交（单批最多 10,000 请求 / 256 MB），同样提供 **50% Batch 折扣**，且可与 **Prompt Caching 叠加**——在多语种出海场景下，将共享的品牌风格指南、通用术语表和 Few-shot 示例放在缓存前缀中，可实现双重降本。  
       来源：[Claude Platform Docs — Batch Processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)

2. **回译（Back-Translation）与无参考质量评估（QE）**
   - **无参考神经质量评估（Reference-Free QE）**：出海翻译往往没有现成的目标语人工参考译文（Ground Truth），业界主流使用无参考 QE 模型直接基于 `(source, hypothesis)` 评估质量：
     - **Unbabel COMET / CometKiwi**：提供参考译文无关的 QE 模型（如 `wmt22-cometkiwi-da`），输出与人类直觉（DA/MQM）高度相关的句级质量分。  
       来源：[GitHub — Unbabel/COMET](https://github.com/Unbabel/COMET)
     - **xCOMET**：在句级评分之外，进一步定位译文中的**错误片段（Error Spans）**并按 MQM（Multidimensional Quality Metrics）体系划分 `minor` / `major` / `critical` 严重等级，可直接用于“低分定位与高亮”。  
       来源：[arXiv:2310.10482 — xCOMET](https://arxiv.org/abs/2310.10482)
     - **Google MetricX**：开源指标家族（如 MetricX-24），支持有参考与无参考（QE）混合模式（以错误率为导向，分值越低质量越高）。  
       来源：[GitHub — google-research/metricx](https://github.com/google-research/metricx)
   - **回译语义一致性校验（Round-Trip / Back-Translation Consistency）**：将目标语译文回译为源语言（建议使用异构模型或独立 NMT 以减少同源模型盲区），再通过句向量余弦相似度或高阶 LLM 对比“原文 vs 回译文”的核心语义、实体与否定/条件逻辑是否保留。
     - **工程注意**：回译擅长发现**漏译、错译、数值/否定词反转与幻觉**，但回译模型可能“自动脑补”目标语中的语法生硬或语域错误（如敬语不当）；因此推荐将**回译一致性校验与目标语侧 QE（CometKiwi/xCOMET）或 LLM-as-a-Judge 联合使用**，并仅对低分/分歧样本触发人工抽检。

---

## 2. 成熟开源/商业 Prompt 评估与可观测工具对比机制

### 2.1 主流工具评估与对比机制横评

| 工具/规范 | 定位与开源属性 | 评估分层机制 | 对比与 Diff 机制 | 人工抽检与校准闭环 |
| :--- | :--- | :--- | :--- | :--- |
| **Langfuse** | 开源 LLM 可观测与评估平台（可自托管） | Code Evaluators + Managed **LLM-as-a-Judge**（支持在线生产 Trace 采样与离线 Dataset Experiments；分数支持 numeric / categorical / boolean） | 支持选定 Baseline 对比 Experiment 运行；按 Score Delta 筛选失败样本（如 `score < 0.2`），内置 **Prompt Version Diff** 与 **Peek Trace** 联动 | **Annotation Queues** 供领域专家标注与填写修正译文；**Score Analytics** 直接计算人机一致性（Cohen's Kappa、Pearson、Spearman、F1、对比热力图），并支持官方 Judge Calibration 工作流 |
| **Promptfoo** | 开源 CLI / 本地优先 Prompt 测试与红队框架 | 声明式 YAML **分层 Assertions**：先跑零成本确定性断言（`is-json`、`contains`、`regex`、`latency`、`cost`），再跑 `llm-rubric`、`g-eval` 等模型裁判 | **Matrix 矩阵视图**（Prompts × Models × Test Cases）；支持 *All / Failures / Passes / Errors / Different / Highlights* 过滤及行内/并排 Diff，适合 CI 回归门禁 | 支持本地 Web UI 审阅导出与人工覆盖评分，核心偏向研发自测与 CI/CD 自动化门禁 |
| **Braintrust** | AI 评估与可观测平台（含开源 `autoevals` 库） | **Custom Code** + 开源 **Autoevals** + **LLM-as-a-Judge** Scorers（统一归一化到 `0–1` 或分类标签；离线与在线可复用同一 Scorer） | **不可变 Experiments** 对比；支持固定 Baseline（或自动关联同 Git 分支最新实验）、自定义 Comparison Key、按 **Regressions 数量排序**，以及 3 种 **Diff 模式** | 原生 **Human Review** 队列与多人盲评（未评分 Reviewer 默认空白表单防锚定效应）；数值分自动求均值，文本/期望输出可指定某条评审为 Base Span Value |
| **OpenAI Evals / Anthropic Evals 实践** | 基准框架与工程方法论 | OpenAI Graders 支持 `string_check`、`text_similarity`、`score_model`、`python`（`{{ item.* }}` / `{{ sample.* }}` 模板）；Anthropic 明确划分 **Code-based / Model-based / Human** 三类 Grader | 强调 **Capability Evals**（能力爬坡）与 **Regression Evals**（回归门禁，接近 100% 通过）分离，并引入 **`pass@k` vs `pass^k`** 衡量非确定性多次试验稳定性 | Anthropic 强调必须定期人工阅读原始 **Transcript（Execution Trace）**，防止僵化 Grader 误杀合法输出或漏掉作弊模式 |

**一手文档参考**：
- Langfuse Evaluation Overview / LLM-as-a-Judge / Annotation Queues / Compare Experiments / Score Analytics / Calibration：
  - [Langfuse — Evaluation Overview](https://langfuse.com/docs/evaluation/overview.md)
  - [Langfuse — LLM-as-a-Judge](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge.md)
  - [Langfuse — Annotation Queues](https://langfuse.com/docs/evaluation/evaluation-methods/annotation-queues.md)
  - [Langfuse — Compare Experiments](https://langfuse.com/docs/evaluation/experiments/compare-experiments.md)
  - [Langfuse — Score Analytics](https://langfuse.com/docs/evaluation/scores/score-analytics.md)
  - [Langfuse — Calibrate LLM-as-a-Judge](https://langfuse.com/guides/llm-as-a-judge-calibration-skill.md)
- Promptfoo Assertions / LLM Rubric / G-Eval / Web UI：
  - [Promptfoo — Assertions & Expected Outputs](https://www.promptfoo.dev/docs/configuration/expected-outputs/)
  - [Promptfoo — llm-rubric](https://www.promptfoo.dev/docs/configuration/expected-outputs/model-graded/llm-rubric/)
  - [Promptfoo — G-Eval](https://www.promptfoo.dev/docs/configuration/expected-outputs/model-graded/g-eval/)
  - [Promptfoo — Web UI](https://www.promptfoo.dev/docs/usage/web-ui/)
- Braintrust Scorers / LLM-as-a-Judge / Compare Experiments / Human Review：
  - [Braintrust — Writing Scorers](https://docs.braintrust.dev/best-practices/scorers.md)
  - [Braintrust — LLM-as-a-Judge Evaluators](https://docs.braintrust.dev/evaluate/llm-as-a-judge.md)
  - [Braintrust — Compare Experiments](https://docs.braintrust.dev/evaluate/compare-experiments.md)
  - [Braintrust — Multiple Reviewers](https://docs.braintrust.dev/annotate/human-review/multiple-reviewers.md)
- OpenAI & Anthropic 官方评估资料：
  - [GitHub — openai/evals](https://github.com/openai/evals)
  - [OpenAI API Docs — Evals](https://developers.openai.com/api/docs/guides/evals)
  - [OpenAI API Docs — Graders](https://developers.openai.com/api/docs/guides/graders)
  - [Anthropic Engineering — Demystifying evals for AI agents](https://www.anthropic.com/research/demystifying-evals-for-ai-agents)

---

### 2.2 三层评分架构与 Rubric 防偏设计

综合上述工具与学术一手研究，成熟的三层评分架构应遵循**“先确定性门禁、再结构化裁判、最后人工校准闭环”**的漏斗原则：

#### 第一层：确定性规则校验（Deterministic Pre-Flight Gate）
- **职责**：用零 Token 成本、毫秒级、100% 可复现的代码规则拦截硬错误，作为 LLM-as-a-Judge 的前置门禁（Promptfoo 推荐的 Layered Assertions 模式）。
- **典型规则（尤其针对出海翻译与结构化 Workflow）**：
  - 结构与语法校验：`is-json` / JSON Schema 校验、XML/HTML 标签闭合、ICU MessageFormat AST 合法性；
  - 约束校验：占位符集合 100% 一致性、术语表词条命中率（含词形/大小写容差规则）、禁译词未改动、字符长度膨胀比阈值（如 `len(mt)/len(src)` 异常预警）；
  - 运行预算断言：`latency` 与 `cost` 阈值。

#### 第二层：LLM-as-a-Judge Rubric 设计与偏差缓解
- **学术界已证实的裁判偏差（Primary Evidence）**：
  1. Zheng et al. (2023) *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena* 系统揭示了三大核心偏差：**位置偏差（Position Bias）**（成对比较时偏好首个或第二个位置）、**长度/啰嗦偏差（Verbosity Bias）**（偏好更长、看似详尽但未必更准的回答）、**自评/同族偏好偏差（Self-Enhancement Bias）**（裁判模型倾向给自己或同家族模型打高分），以及复杂推理验证能力有限的问题。  
     来源：[arXiv:2306.05685 — MT-Bench & Chatbot Arena](https://arxiv.org/abs/2306.05685)
  2. Ye et al. (2024) *Justice or Prejudice? Quantifying Biases in LLM-as-a-Judge* 通过 CALM 框架进一步量化了多达 12 类裁判偏差，证实即便是强模型在特定任务上仍存在显著系统性偏倚。  
     来源：[arXiv:2410.02736 — Justice or Prejudice?](https://arxiv.org/abs/2410.02736)
  3. Liu et al. (2023) *G-Eval* 提出**自动生成 CoT 评估步骤 + 表单填充（Form-Filling）+ 输出分数 Token 对数概率加权（$\text{Score} = \sum P(s_i)\cdot s_i$）**，在摘要与对话等 NLG 任务上显著提升与人类评审的相关性。  
     来源：[arXiv:2303.16634 — G-Eval](https://arxiv.org/abs/2303.16634)

- **工程化防偏 Rubric 设计规范**：
  1. **位置偏差缓解（Pairwise A/B 对比场景）**：执行**双向位置交换（Position Swapping）**——同一对样本分别以 `(A, B)` 和 `(B, A)` 送入裁判；仅当两次胜负一致时记为胜出，不一致则归为 `Tie`（平局）或升级人工复核。
  2. **长度偏差缓解**：在 Rubric 中显式写入反冗余约束（例如：“不得仅因译文更长或添加解释性修饰而给高分；若出现原文不存在的增译或废话应扣分”）；并在翻译场景引入长度比例上下文提示。
  3. **自评偏差缓解**：采用**异构跨家族高阶裁判**（例如被测节点用 GPT/DeepSeek/Qwen，高阶裁判优先交叉使用另一家族的强模型如 Claude/Gemini/GPT 强推理版本，并隐藏被测模型名称）。
  4. **低粒度/二元维度拆解优于模糊 1–10 分**：Braintrust 与 Langfuse 最佳实践均指出，连续 1–10 整数评分边界模糊、方差大；应将整体评价拆解为独立的**二元（Pass/Fail）**或**3 档有序标签**（如翻译场景借鉴 MQM：`Pass` / `Minor Issue` / `Critical Error`），再由系统聚合总分；若需细粒度连续分，可采用 G-Eval 的 Token Logprob 加权或结构化扣分项清单。
  5. **先思考、后打分 + 参考引导（Reference-Guided）**：强制裁判模型在 JSON Schema 中先输出 `reasoning` / `error_spans` 字段，再输出 `score` / `verdict`；当数据集存在参考答案或术语表时，将其作为显式锚点（`{{expected}}`、`{{glossary}}`）输入裁判。

#### 第三层：人工抽检（Spot-Check）与裁判校准流程
1. **分层抽样策略（Stratified Sampling）**：
   - **高风险全量/高比例抽检**：确定性门禁通过但 LLM 裁判低分、A/B 位置交换结果冲突、新版本相对 Baseline 出现 **Regression** 的样本；
   - **随机基线抽检**：从高分/通过样本中按固定比例（如 5%–10%）随机抽取，用于监控裁判的假阳性（漏报）。
2. **盲审机制与多评审汇聚**：借鉴 Braintrust Human Review 设计，人工评审界面默认隐藏模型身份与自动评分，避免**锚定效应（Anchoring Bias）**；多人标注同一 Trace 时，各自写入独立 Review Span，数值分求平均，修正后的 Expected Output 需显式“提升为基准值”。  
   来源：[Braintrust — Review with multiple reviewers](https://docs.braintrust.dev/annotate/human-review/multiple-reviewers.md)
3. **一致性度量与周期性重校准（Calibration）**：
   - **拒绝仅看 Raw Agreement**：在通过率 90% 的不平衡数据上，“全判 Pass”也能拿到 90% 原始一致率却毫无价值。应借鉴 Langfuse Score Analytics，使用 **Cohen's Kappa ($\kappa$)**、**F1**、**TPR / TNR**（分类/二元场景）以及 **Spearman / Pearson / MAE**（连续分场景）来评估 Judge 与人类专家的一致性（业界通常以 $\kappa \ge 0.6\text{–}0.7$ 作为可信门槛）。
   - **Bad-case 回流黄金集**：将人工修正后的高质量样本沉淀为版本化黄金测试集（Golden Dataset），用于后续新 Prompt 或新裁判模型的回归验证与定期重校准。

---

## 3. Prompt 版本 Diff、Output Diff 与汇总报告的业界设计模式

### 3.1 Prompt 版本 Diff 设计模式

参考 Langfuse Prompt Management 与 Braintrust Prompts Versions 的实践：
- **不可变版本 + 语义标签（Versions & Labels）**：每次保存生成不可变版本号（如 `v1`, `v2` 或内容哈希），同时支持环境/实验标签（`production`、`staging`、`candidate`）动态切换而无需改代码。  
  来源：[Langfuse — Prompt Version Control](https://langfuse.com/docs/prompt-management/features/prompt-version-control.md) | [Braintrust — Version prompts](https://docs.braintrust.dev/evaluate/prompts/versions.md)
- **双维度 Diff（不仅对比文本，还对比配置）**：
  1. **Prompt 文本/消息 Diff**：支持行内（Inline）与并排（Side-by-side）的 System/User/Few-shot 消息增删高亮，识别模板变量 `{{var}}` 变更；
  2. **绑定配置 Diff**：明确展示 `model`、`temperature`、`max_tokens`、`response_format`（JSON Schema）、绑定工具（Tools）以及关联术语表/上下文规则的差异，防止“只改了模型或温度却误以为是文案改动导致效果变化”。

### 3.2 Output Diff 设计模式

综合 Braintrust Compare Experiments、Langfuse Compare Experiments 与 Promptfoo Web UI：
1. **稳定行对齐键（Comparison Key）与缺失保护**：
   - 默认按 Dataset Row ID 或标准化输入字段对齐（Braintrust 支持自定义复合 Comparison Key，如 `[input.source, metadata.locale]`）；
   - Langfuse 特别强调：**缺失样本不等于通过（A missing case is not a passing case）**——若候选版本因超时或异常漏跑某些行，必须在对比报告中显式标记为 `Missing / Error` 而非计入均值分母之外。  
     来源：[Langfuse — Compare Experiments](https://langfuse.com/docs/evaluation/experiments/compare-experiments.md)
2. **三种核心 Diff 视图模式**（Braintrust 模式）：
   - `Baseline → Candidate`（对比新版相对基线改了什么）；
   - `Candidate → Baseline`（反向高亮）；
   - `Expected → Output`（单次运行内对比生成结果与参考标准答案）；
   - 同时支持**表格行内展开（Inline Sub-rows）**、**详情抽屉字符/词级 Diff（Side-by-side / Inline）**，以及 Promptfoo 式的 **Prompt × Test Case 矩阵视图（Matrix View）**，在多 Prompt 版本或多语种横向对比时尤为高效。  
     来源：[Braintrust — Compare Experiments](https://docs.braintrust.dev/evaluate/compare-experiments.md) | [Promptfoo — Web UI](https://www.promptfoo.dev/docs/usage/web-ui/)
3. **回归优先的分诊过滤（Regression-First Triage）**：
   - 支持一键按 **Regressions 数量排序**，以及按 `Regressions`、`Improvements`、`Failures`、`Errors`、`Different` 和得分阈值（如 `score < 0.2`）过滤，并能直接一键下钻到对应 **Execution Trace**（Langfuse Peek View）定位是上游检索/术语注入失败还是模型本身生成失误。

### 3.3 耗时 / Token 费用 / 准确率汇总报告的关键指标

为避免单一均值掩盖局部退化（例如“50% 平均准确率”可能来自原本通过的核心用例退化 + 边缘用例通过），业界评估汇总报告应包含以下四个指标维度：

| 指标维度 | 核心指标项 | 设计原因与业界实践 |
| :--- | :--- | :--- |
| **质量与准确率（Accuracy & Regression）** | • 综合 Pass Rate（全部断言通过率）<br>• 分维度得分均值与分布直方图<br>• **Score Delta（`+ / -`）**<br>• **Regressions / Improvements 计数**<br>• 多次试验稳定性：**`pass@k` 与 `pass^k`**<br>• 翻译专项：确定性门禁通过率、QE 分数（COMETKiwi/xCOMET/MetricX）、MQM 严重度分布 | Braintrust 在 Summary Table 中将各实验列为竖栏、各评分与指标列为横行并高亮最优/最差；Langfuse 与 Anthropic 均强调必须显式呈现 Regression 计数与多轮采样稳定性，配合 CI 门禁阻断回归（[Langfuse CI/CD](https://langfuse.com/docs/evaluation/experiments/experiments-ci-cd.md)）。 |
| **耗时（Latency & Throughput）** | • **分位数耗时：`p50`、`p95`、`p99`**（而非仅平均值）<br>• 首字延迟 **TTFT**（流式交互场景）<br>• Workflow 端到端总耗时 vs 单 Node 耗时拆解<br>• Tokens/sec 吞吐量 | 长尾延迟（p95/p99）直接决定批量翻译与在线体验稳定性；Braintrust 支持在 Traces 与 Spans 两级分别查看耗时归因（[Braintrust Interpret Results](https://docs.braintrust.dev/evaluate/interpret-results.md)）。 |
| **Token 与费用（Tokens & Cost）** | • Input / Output / Cached / Reasoning Tokens 分类统计<br>• **任务推理成本 vs 裁判评估成本分离统计**<br>• 单条平均成本 & 全量预估成本（如对比同步 API vs Batch 50% 折扣）<br>• 上下文阶梯定价感知 | Langfuse 与 Braintrust 均将 Token 与预估成本作为对比视图的一等公民列（与得分并列进行性价比权衡）；分离 Judge 成本有助于控制“三层评估”自身开销。 |
| **可靠性与元数据切片（Reliability & Segmentation）** | • 解析/Schema 失败率、HTTP 429/超时率<br>• 按元数据切片对比（如按 `locale`、`domain`、`string_length` 分组聚合） | Braintrust 强调通过 `Group by` 元数据切片防止“总体均分上升但某一小语种或核心业务线大幅退化”的辛普森悖论。 |

---

## 4. 对 PromptDone 架构设计的落地映射建议

结合 [PromptDone 领域术语表](../../GLOSSARY.md)，上述最佳实践可直接映射为平台核心能力模块：

1. **Dataset（数据集）与翻译预处理**：
   - 上传清洗后的源语料形成不可变版本 **Dataset** 时，自动识别并提取占位符/ICU 骨架存入结构化字段；支持绑定版本化术语表（Glossary）作为 Dataset 关联资产或参考上下文。
2. **Workflow（工作流）与 Node（节点）编排**：
   - 将出海翻译建模为多 **Node** 组成的 **Workflow** DAG：例如 `节点 1：动态术语检索 + 占位符掩码` → `节点 2：目标语翻译（按语种扇出并受控并发）` → `节点 3：确定性占位符/ICU 校验与自愈重试` → `节点 4：回译或高阶模型 Reflection/QE`。
   - 对大批量 Dataset 支持同步受控并发队列与异步 Batch API 执行模式切换，并将每条翻译过程完整记录为 **Execution Trace**。
3. **Tuning Session（调优会话）的三层评估与对比**：
   - 在 **Tuning Session** 中把评估拆为三层漏斗：**Deterministic Gate（免费硬校验）** → **LLM-as-a-Judge（异构高阶模型 + 二元/MQM 分拆 Rubric + 位置交换/防啰嗦约束）** → **Human Spot-Check（分层抽样盲审队列 + Cohen's Kappa 校准面板）**。
   - **对比报告视图**：提供 Prompt（文本 + 模型参数）Diff、基于 Comparison Key 对齐的 Output Diff（支持 Baseline/Candidate/Expected 切换与 Regression 优先排序），以及包含 **Pass Rate / Regressions 数 / `p50` & `p95` Latency / Task vs Judge Cost / 按语种切片统计** 的汇总表。
4. **Context Bundle（诊断上下文包）导出**：
   - 当某次对比报告发现占位符修复逻辑缺陷、术语匹配漏召回或某语种集中 Regression 时，一键将相关 Prompt Diff、失败样本、Expected vs Actual 译文 Diff 及 **Execution Trace** 打包为 **Context Bundle**，直接交付 Codex / Antigravity 修复代码或调优规则。
