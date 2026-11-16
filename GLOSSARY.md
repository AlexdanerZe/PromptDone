# PromptDone

A Prompt tuning and data-cleaning workbench for preparing datasets, running multi-node Prompt workflows, comparing versions, and exporting bad-case context to external coding harnesses.

## Language

**Dataset**:
A logical, named collection of versioned data snapshots produced from uploaded Excel, CSV, or JSON files.
_Avoid_: Data source, raw table, test file

**Dataset Version**:
An immutable, numbered snapshot (`v1`, `v2`, ...) of cleaned and schema-validated **Dataset Records** bound to **Workflow** runs and **Execution Traces**.
_Avoid_: Dataset snapshot, data revision

**Dataset Record**:
A single standardized item inside a **Dataset Version** or **Import Draft**, mapped into `Input` variables, optional `Expected Output`, `Comparison Key`, and `Metadata`.
_Avoid_: Row, test case, sample

**Import Draft**:
A mutable staging workspace created during file upload, append, or re-clean where column mappings, JSON repairs, and row exclusions are finalized before publishing a **Dataset Version**.
_Avoid_: Staging table, upload cache

**Prompt**:
A reusable instruction asset comprising a mutable draft and immutable published versions, defining message templates, default model parameters, and output format constraints.
_Avoid_: System message, template

**Prompt Version**:
An immutable, numbered snapshot (`v1`, `v2`, ...) of a **Prompt** that freezes its message templates, model configuration, and output schema for locking by a **Node**.
_Avoid_: Prompt snapshot, prompt revision

**Node**:
A single step inside a **Workflow** that binds to a published **Prompt Version** or live Draft and executes single-turn or sequential multi-turn inference to transform input variables into structured or text output.
_Avoid_: Step, stage, module

**Workflow**:
A directed acyclic graph (DAG) of connected **Nodes** where upstream outputs map to downstream inputs for single-item, random-sample, or batch evaluation. The UI label “调优任务” names this same concept.
_Avoid_: Pipeline, flow, chain

**Workflow Run**:
An execution snapshot of a **Workflow** that freezes its topology, each **Node**'s active Prompt and parameters, the bound **Dataset Version**, selected records, and resulting **Execution Traces**. A **Tuning Session** may group runs, but is not required to create one.
_Avoid_: Batch job, test run, run history item

**Execution Trace**:
The recorded per-item execution log of a **Dataset Record** (or a `(Dataset Record, target_lang)` pair in multi-language matrix runs) within a **Workflow Run**, capturing DAG node execution details, turn-by-turn rendered messages, raw responses, structured parsing status, cache hits, latency, and token cost.
_Avoid_: Chat log, conversation history, run trace

**Tuning Session**:
An iterative optimization workspace recording Prompt draft edits, AI-assisted suggestions, and evaluation comparisons across runs.
_Avoid_: Prompt chat, optimization thread

**Tuning Suggestion**:
A persisted AI-assisted optimization record inside a **Tuning Session** capturing selected Bad Cases, human diagnostic notes, root-cause analysis, diffable **Prompt** rewrite proposals, follow-up refinement turns, adoption status, and linked comparison **Workflow Runs**.
_Avoid_: AI prompt fix, tuning chat message

**Context Bundle**:
A portable diagnostic package containing Prompt definitions, failing samples, expected vs. actual outputs, and evaluation summaries formatted for external harnesses such as Codex or Antigravity.
_Avoid_: Export file, error dump, prompt packet

**Workspace**:
An isolated team or project space that scopes **Datasets**, **Prompts**, **Workflows**, **Tuning Sessions**, and **Model Provider** configurations.
_Avoid_: Tenant, project, organization

**Model Provider**:
A configured LLM service endpoint within a **Workspace** that encapsulates credentials, rate-limit settings, custom model pricing overrides, and structured-output capability flags.
_Avoid_: API vendor, gateway config, LLM key

**Eval Suite**:
A reusable **Workspace**-level evaluation configuration defining deterministic rules and, when configured, LLM-as-a-Judge rubrics and weights, bound to a **Workflow** or a specific **Node**.
_Avoid_: Eval config, scoring template, test suite

**Evaluation Run**:
An immutable scoring pass attached to a **Workflow Run** that freezes the applied **Eval Suite** rules, optional judge configuration, and resulting automatic scores. Human judgments remain separate **Trace Annotations**.
_Avoid_: Score run, judge batch, eval job

**Trace Annotation**:
A human judgment and diagnostic note for a specific **Node** execution within an **Execution Trace**, editable independently of frozen model outputs and automatic evaluation results.
_Avoid_: Output edit, automatic score override

**Prompt Review**:
An on-demand static diagnostic pass performed by a high-tier model against a **Prompt** draft or **Prompt Version** using a review rubric, producing severity-ranked findings and diffable rewrite suggestions.
_Avoid_: Prompt lint, prompt audit, prompt check

**Glossary**:
A **Workspace**-level multilingual terminology table mapping source terms to target-locale translations, `do_not_translate` flags, case-sensitivity rules, and usage notes for dynamic injection and compliance checking.
_Avoid_: Word list, dictionary, term sheet

**Style Guide**:
A **Workspace**-level localization specification defining tone, register, honorifics, formatting conventions, and forbidden expressions per locale (with language-prefix fallback) that attaches to a **Node** and freezes into **Workflow Run** snapshots.
_Avoid_: Tone prompt, brand doc, localization memo
