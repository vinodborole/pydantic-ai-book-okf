---
type: Web Page
title: Third-Party Capabilities | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/third-party
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Third-Party Capabilities

[Capabilities](/docs/ai/capabilities/overview) are the recommended way for third-party packages to extend Pydantic AI, since they can bundle tools with hooks, instructions, and model settings. See [Extensibility](/docs/ai/guides/extensibility) for the full ecosystem, including [third-party toolsets](/docs/ai/tools-toolsets/toolsets#third-party-toolsets) that can also be wrapped as capabilities.

Many of the use cases below are also covered by first-party capabilities in Pydantic AI itself or [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/), the official capability library. Where that’s the case, we point to the built-in option first, and list the community packages as alternatives.

For model-owned task planning and progress tracking, [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) ships [ Planning](https://pydantic.dev/docs/ai/harness/planning/), a cache-friendly self-updating task plan. As a community alternative with subtask, dependency, and PostgreSQL persistence support:

- `pydantic-ai-todo`- `TodoCapability`with- `add_todo`,- `read_todos`,- `write_todos`,- `update_todo_status`, and- `remove_todo`tools. Supports subtasks, dependencies, and PostgreSQL persistence. Also available as a lower-level- `TodoToolset`.

Pydantic AI has [built-in compaction](/docs/ai/capabilities/compaction) — provider-native APIs and model-agnostic history summarization — and [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/compaction/) adds a full menu of compaction strategies. As a community alternative:

- `summarization-pydantic-ai`- `ContextManagerCapability`(real-time token tracking, auto-compression at a configurable threshold, and large tool-output truncation);- `SummarizationCapability`(LLM-powered history compression);- `SlidingWindowCapability`(zero-cost message trimming);- `LimitWarnerCapability`(injects a finish-soon hint before hard context limits). Also available as standalone- `history_processors`:- `SummarizationProcessor`,- `SlidingWindowProcessor`, and- `LimitWarnerProcessor`.

Pydantic AI supports [multi-agent patterns](/docs/ai/guides/multi-agent-applications) directly, and [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/subagents/) ships [ SubAgents](https://pydantic.dev/docs/ai/harness/subagents/) for delegating self-contained tasks to named child agents. As a community alternative:

- `subagents-pydantic-ai`- `SubAgentCapability`adds tools for multi-agent delegation:- `task`(spawn a subagent),- `check_task`,- `wait_tasks`,- `list_active_tasks`,- `soft_cancel_task`,- `hard_cancel_task`, and- `answer_subagent`. Supports sync, async, and auto-execution modes, nested subagents, and runtime agent creation. Also available as a lower-level toolset via- `create_subagent_toolset`.

[Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/guardrails/) provides input and output guardrails that validate or block requests and responses, and Pydantic AI enforces usage, token, and request limits via [ UsageLimits](/docs/ai/core-concepts/agent#usage-limits). As a community alternative bundling several ready-made shields, including USD cost tracking:

- `pydantic-ai-shields`- `CostTracking`(tracks token usage and USD cost per run, raises- `BudgetExceededError`on budget overrun);- `ToolGuard`(block or require approval for specific tools);- `InputGuard`and- `OutputGuard`(custom sync or async validation functions);- `PromptInjection`,- `PiiDetector`,- `SecretRedaction`,- `BlockedKeywords`, and- `NoRefusals`content shields.

[Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) ships sandboxed [ FileSystem](https://pydantic.dev/docs/ai/harness/filesystem/) and 

[capabilities, plus](https://pydantic.dev/docs/ai/harness/shell/)

`Shell`[for running tool calls as sandboxed Python. As a community alternative:](https://pydantic.dev/docs/ai/harness/code-mode/)

`CodeMode`- `pydantic-ai-backend`- `ConsoleCapability`registers- `ls`,- `read_file`,- `write_file`,- `edit_file`,- `glob`,- `grep`, and- `execute`tools with a fine-grained permission system. Backends include- `StateBackend`(in-memory, for testing),- `LocalBackend`(real filesystem),- `DockerSandbox`(isolated container execution), and- `CompositeBackend`(routing across backends). Also available as a lower-level- `ConsoleToolset`.

Pydantic AI supports [Agent Skills natively](/docs/ai/capabilities/on-demand#loading-skills-from-markdown-files) through [on-demand capabilities](/docs/ai/capabilities/on-demand), which collapse a skill to a one-line catalog entry until the model loads it. As a community alternative:

- `pydantic-ai-skills`- `SkillsCapability`implements Agent Skills support with progressive disclosure (load skills on-demand to reduce tokens). Supports filesystem and programmatic skills; compatible with- [agentskills.io](https://agentskills.io).

Capabilities for querying and analyzing structured data help agents answer questions over files and databases:

- `pydantic-ai-chdb`- `ChDBCapability`gives agents analytical SQL over local files (Parquet/CSV/JSON), object storage, and remote databases with- [chDB](https://clickhouse.com/docs/en/chdb), the in-process ClickHouse engine — the engine itself needs no server or connection string to run (remote sources are reached via ClickHouse table functions, which take their own credentials). Registers- `run_select_query`(read-only ClickHouse SQL with parameter binding),- `list_databases`,- `list_tables`,- `describe_table`,- `get_sample_data`,- `list_functions`, and- `attach_file`(opt-in writable sessions) tools plus schema-first instructions. Sessions default to the engine-level- `readonly=2`setting with capped results, and typed engine errors are mapped to- `ModelRetry`- [agent specs](/docs/ai/core-concepts/agent-spec)out of the box, so it can be loaded via- `from_spec`- `Agent.from_spec`- [toolset](/docs/ai/tools-toolsets/toolsets)via- `ChDBCapability(...).get_toolset()`

To add your package to this page, open a pull request.

To publish your own capability package, see [Publishing capabilities](/docs/ai/capabilities/custom#publishing-capabilities) and [Extensibility](/docs/ai/guides/extensibility).

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/third-party
