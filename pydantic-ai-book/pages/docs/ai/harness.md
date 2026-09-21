---
type: Web Page
title: Pydantic AI Harness | Pydantic Docs
description: 'Your agent''s favorite harness, built on Pydantic AI: 30+ capabilities
  and complete agents assembled from them, from a coding agent to your own custom
  stack.'
resource: https://pydantic.dev/docs/ai/harness
timestamp: '2026-09-21T12:25:24.826293+00:00'
---

# Pydantic AI Harness

*Your agent’s favorite harness, built on Pydantic AI*

**Pydantic AI Harness** is the official [capability](/docs/ai/capabilities/overview/) and harness library for [Pydantic AI](/docs/ai/). Every Pydantic AI agent already has a light harness: the typed agent loop, [any model](/docs/ai/models/overview/), your own tools, structured output. For simple agents that’s enough. But set an agent loose on complex, long-running work (fix a codebase, research a question, run for hours unattended) and what it needs around the model grows: a [workspace](/docs/ai/harness/filesystem/) to act in, a [plan](/docs/ai/harness/planning/) it keeps current, [memory](/docs/ai/harness/memory/) that carries across sessions, [sub-agents](/docs/ai/harness/subagents/) to hand work to, [context management](/docs/ai/harness/compaction/) that holds up in hour ten, and [durable execution](/docs/ai/capabilities/durable_execution/overview/) that survives a restart. **Pydantic AI Harness** ships that harness.

Everything here is one primitive: a [capability](/docs/ai/capabilities/overview/), a self-contained unit of agent behavior you add to `capabilities=[...]` on any agent. There are [30+ of them](#capabilities), and complete agents like [Coder](/docs/ai/harness/coder/) and [Researcher](/docs/ai/harness/researcher/) are themselves capabilities combined: they come apart the way they went together. Snap on a single block, compose your own stack, or start from the whole coding agent and take it apart later.

Install with [`uv`](https://docs.astral.sh/uv/):

```
from pydantic_ai import Agent
from pydantic_ai_harness import Coder
agent = Agent('anthropic:claude-fable-5', capabilities=[Coder()])
result = agent.run_sync('Find out why tests/test_parser.py fails and fix the bug it caught.')
print(result.output)
#> Found it: `parse()` returned None on empty input instead of raising. Fixed in src/parser.py; tests pass now.
```
Coder provides six tools: `read_file`, `write_file`, `edit_file`, `list_files`, `grep`, and `shell`, plus repository context and context controls. Shell commands are unrestricted and can persist beyond individual runs. Default instructions guide autonomous investigation, editing, and verification; pass `instructions=` to add your own guidance.

Every model works: swap the string for [any provider’s](/docs/ai/models/overview/). Need more? Add capabilities to the list; here’s the same coder on `gpt-5.6-sol`, with web search and cross-session memory:

```
from pydantic_ai import Agent
from pydantic_ai.capabilities import WebSearch
from pydantic_ai_harness import Coder, Memory
from pydantic_ai_harness.memory import FileStore
agent = Agent(
    'openai:gpt-5.6-sol',
    capabilities=[
        Coder(),
        WebSearch(),  # look up docs and error messages on the web
        Memory(FileStore('.agent-memory')),  # remembers across sessions
    ],
)
```
[Skills](/docs/ai/harness/skills/) (your `SKILL.md` procedures, loaded on demand; point it at a `skills/` directory and add the `skills` extra), [Web Fetch](/docs/ai/capabilities/web-fetch/), [Guardrails](/docs/ai/harness/guardrails/), and [Dynamic Workflow](/docs/ai/harness/dynamic-workflow/) slot in the same way; the [Coder page](/docs/ai/harness/coder/#composition) lists what pairs well.

`Coder` is a regular combined capability: [`FileSystem`](/docs/ai/harness/filesystem/) with five of its tools and content hashes off, [`Shell`](/docs/ai/harness/shell/) with its persistent `shell` tool and no allowlist, [`RepoContext`](/docs/ai/harness/repo-context/), [`ClearToolResults` and `WarnNearLimits`](/docs/ai/harness/compaction/), and a bounded [`ToolOutputLimits`](/docs/ai/harness/tool-output-limits/), plus its default instructions and JSON argument repair. Use it whole, or build the same agent from those capabilities to change any setting; the [Coder page](/docs/ai/harness/coder/) lists the exact configuration.

```
from pydantic_ai import Agent
from pydantic_ai_harness.coder import Coder
agent = Agent(
    'anthropic:claude-fable-5',
    name='coder',
    capabilities=[Coder('.')],
)
```
See the Coder documentation for tool signatures, persistent shell lifecycle, and migration from the previous planning/delegation composition.

Every capability is a self-contained unit you drop into `capabilities=[...]`, and they all compose, with each other and with your own. Some come with [`pydantic-ai`](/docs/ai/) itself, the rest with this package; the **Package** column says which. 50+ in all, grouped by what they give your agent:

Complete agent stacks as regular combined capabilities: one import gives you a working agent, and you can take either apart into the blocks below.

| Harness | Package | What it provides | 
|---|---|---|
| [Coder](/docs/ai/harness/coder/) | Harness | Six coding tools, persistent shell commands, autonomous guidance, and context controls | 
| [Researcher](/docs/ai/harness/researcher/) | Harness | A complete web-research stack: search, page fetching, a delegated sub-researcher, and bounded tool output | 

The workspace the agent acts in: the files it edits and the commands it runs, local or isolated.

| Capability | Package | What it does | 
|---|---|---|
| [FileSystem](/docs/ai/harness/filesystem/) | Harness | Read, write, edit, list, and search files under a root, with opt-in ripgrep tools; path-traversal and symlink safe, secrets read-only | 
| [Shell](/docs/ai/harness/shell/) | Harness | Command execution with allowlists, denylists, timeouts, credential-stripping, and opt-in commands that outlive the run | 
| [Modal Sandbox](/docs/ai/harness/modal-sandbox/) | Harness | Commands and files in an isolated [Modal](https://modal.com) cloud sandbox | 

Connections to systems outside the agent’s workspace, and abilities the provider executes natively.

| Capability | Package | What it does | 
|---|---|---|
| [MCP](/docs/ai/capabilities/mcp/) | Core | Connect any MCP server’s tools; local by default, provider-native connectors opt-in | 
| [Image Generation](/docs/ai/capabilities/image-generation/) | Core | Generate and edit images; provider-native where supported, sub-agent fallback elsewhere | 
| [StackOne](/docs/ai/harness/stackone/) | Harness | Act on linked SaaS accounts (HRIS, ATS, CRM, …) via [StackOne](https://www.stackone.com) | 
| [LocalStack](/docs/ai/harness/localstack/) | Harness | An emulated AWS environment with AWS CLI tools | 
| [Macroscope](/docs/ai/harness/macroscope/) | Harness | Run a local [Macroscope](https://docs.macroscope.com/cli) code review and hand the findings to the agent | 

Finding and reading things on the open web.

| Capability | Package | What it does | 
|---|---|---|
| [Web Search](/docs/ai/capabilities/web-search/) | Core | Provider-native search where available, local DuckDuckGo fallback everywhere | 
| [Web Fetch](/docs/ai/capabilities/web-fetch/) | Core | Fetch and read URLs, native or local | 
| [X Search](/docs/ai/capabilities/x-search/) | Core | Search X; native on xAI, subagent fallback elsewhere | 
| [Exa Search](/docs/ai/harness/exa-search/) | Harness | Web research via [Exa](https://exa.ai) : excerpted search, full-page reads, opt-in cited deep search | 
| [Exa Agent](/docs/ai/harness/exa-search/) | Harness | Delegate open-ended research to the Exa Agent API | 
| [You.com Search](/docs/ai/harness/youdotcom/) | Harness | Web search and page reads via [You.com](https://you.com) : query-relevant excerpts or full-page markdown | 
| [You.com Research](/docs/ai/harness/youdotcom/) | Harness | Cited answers and multi-step research via the You.com Answer, Research, and Finance Research APIs | 
| [Browser Use](/docs/ai/harness/browser-use/) | Harness | Hand web tasks to an autonomous [browser-use](https://github.com/browser-use/browser-use) agent driving a real browser | 
| [Playwright Browser](/docs/ai/harness/playwright/) | Harness | Drive a real Chromium page yourself: navigate, click, type, read, and inspect what the page did | 

How the agent thinks and divides the work.

| Capability | Package | What it does | 
|---|---|---|
| [Thinking](/docs/ai/capabilities/thinking/) | Core | Provider-adaptive extended thinking at configurable effort | 
| [Planning](/docs/ai/harness/planning/) | Harness | Model-owned task plans with a cache-safe live reminder | 
| [Subagents](/docs/ai/harness/subagents/) | Harness | Delegate self-contained tasks to named child agents | 
| [Dynamic Workflow](/docs/ai/harness/dynamic-workflow/) | Harness | The model orchestrates sub-agents from one Python script: fan-out, chain, vote in a single tool call, with hard `max_agent_calls` budgets | 
| [Advisor](/docs/ai/harness/advisor/) | Harness | Let an executor consult a stronger model mid-run | 

How the agent spends its context window: the difference between an agent that degrades over a long run and one that doesn’t, and between paying for tokens N times or once.

| Capability | Package | What it does | 
|---|---|---|
| [Code Mode](/docs/ai/harness/code-mode/) | Harness | The model writes one Python script that calls many tools inside a [Monty](https://github.com/pydantic/monty) sandbox: one round-trip instead of N, and intermediate results never enter the context window. The answer to tool-call token bloat | 
| [Tool Search](/docs/ai/capabilities/tool-search/) | Core | Load tool definitions on demand instead of carrying hundreds in every prompt | 
| [Compaction](/docs/ai/capabilities/compaction/) | Core | Provider-native compaction on OpenAI and Anthropic; the provider summarizes history server-side | 
| [Compaction](/docs/ai/harness/compaction/) | Harness | Model-agnostic strategies: tool-result clearing, sliding-window trimming, LLM summarization, tiered; all window-relative, with live usage reporting | 
| [Tool Output Limits](/docs/ai/harness/tool-output-limits/) | Harness | Truncate, spill to a queryable file, or summarize oversized tool returns at the source | 
| [Warn On Cache Busts](/docs/ai/harness/warn-on-cache-busts/) | Harness | Detect prompt-cache prefix collapses between requests, from the provider’s own numbers | 

What the agent knows and remembers, loaded when relevant instead of carried in every prompt.

| Capability | Package | What it does | 
|---|---|---|
| [Memory](/docs/ai/harness/memory/) | Harness | A persistent, namespaced notebook: bounded prompt injection, on-demand search; in-memory/file/Postgres stores | 
| [Conversation Search](/docs/ai/harness/conversation-search/) | Harness | BM25 search over stored history, including turns compaction dropped | 
| [Skills](/docs/ai/harness/skills/) | Harness | Load [Agent Skill](/docs/ai/capabilities/on-demand/) (`SKILL.md` ) instructions on demand | 
| [Repo Context](/docs/ai/harness/repo-context/) | Harness | Start runs oriented: `AGENTS.md` /`CLAUDE.md` + repository structure | 
| [Pydantic AI Docs](/docs/ai/harness/pydantic-ai-docs/) | Harness | On-demand Pydantic AI documentation lookup | 

Bounding what the agent may do, and keeping it on-instructions.

| Capability | Package | What it does | 
|---|---|---|
| [Repair Tool Arguments](/docs/ai/harness/repair-tool-arguments/) | Harness | Repair malformed JSON tool arguments before schema validation. | 
| [Guardrails](/docs/ai/harness/guardrails/) | Harness | Validate/block/redact user input, tool calls, tool results, and output, including secret masking and parallel async guards | 
| [Prompt Injection Defender](/docs/ai/harness/prompt-injection-defender/) | Harness | Classify local tool results for indirect prompt injection and optionally withhold high-risk results | 
| [Spend Limits](/docs/ai/harness/spend/) | Harness | Cross-window USD/token budgets and per-response cost tracking, per model and per tenant | 
| [Ask User](/docs/ai/harness/ask-user/) | Harness | Let the model ask the user multiple-choice questions mid-run; you supply the answerer (terminal, web, test) | 
| [Tool approval](/docs/ai/tools-toolsets/deferred-tools/#human-in-the-loop-tool-approval) | Core | Flag tool calls that need human approval before they run | 
| [Handle Deferred Tool Calls](/docs/ai/capabilities/handle-deferred-tool-calls/) | Core | Resolve approval-deferred tool calls programmatically | 
| [System Reminders](/docs/ai/harness/system-reminders/) | Harness | Cache-safe re-injection of guidance mid-run to counter instruction fade | 
| [Trajectory Judge](/docs/ai/harness/trajectory-judge/) | Harness | A second model reviews the live run every N requests over a sliding token window and steers it mid-run | 

| Capability | Package | What it does | 
|---|---|---|
| [Capability Creation](/docs/ai/harness/capability-creation/) | Harness | The agent writes, validates, and persists *new capabilities* during a run, loaded on the next run: self-extension with typed, inspectable units instead of arbitrary code | 

Outside the loop: how runs persist, survive failures, and get observed and configured in production.

| Capability | Package | What it does | 
|---|---|---|
| [Durable execution](/docs/ai/capabilities/durable_execution/overview/) | Core | Runs that survive restarts and failures on [Temporal](/docs/ai/capabilities/durable_execution/temporal/) ,[DBOS](/docs/ai/capabilities/durable_execution/dbos/) , or[Prefect](/docs/ai/capabilities/durable_execution/prefect/) , with[Restate](/docs/ai/capabilities/durable_execution/restate/) ,[Kitaru](/docs/ai/capabilities/durable_execution/kitaru/) , and[Airflow](/docs/ai/capabilities/durable_execution/airflow/) integrations | 
| [AWS Lambda durability](/docs/ai/harness/aws-lambda/) | Harness | Checkpoint model requests and tool calls into AWS Lambda durable function steps | 
| [Step Persistence](/docs/ai/harness/step-persistence/) | Harness | Save, restore, resume ( `continue_run` ), and fork (`fork_run` ) runs; file/SQLite/Mongo backends | 
| [Instrumentation](/docs/ai/capabilities/instrumentation/) | Core | OpenTelemetry GenAI spans for every model and tool call; the raw material for [Logfire](https://pydantic.dev/logfire) traces | 
| [Managed Prompt](/docs/ai/harness/managed-prompt/) | Harness | Back instructions with a [Logfire](https://pydantic.dev/logfire) -managed prompt; version and roll out without redeploying | 
| [Thread Executor](/docs/ai/capabilities/thread-executor/) | Core | Run sync tools on a shared thread pool | 

Core also ships loop-customization capabilities for production servers: [Select Model](/docs/ai/capabilities/select-model/), [Resolve Model ID](/docs/ai/capabilities/resolve-model-id/), [Prepare Tools / Prepare Output Tools](/docs/ai/capabilities/prepare-tools/), [Prefix Tools](/docs/ai/capabilities/prefix-tools/), [Set Tool Metadata](/docs/ai/capabilities/set-tool-metadata/), [Include Tool Return Schemas](/docs/ai/capabilities/include-tool-return-schemas/), [Process History](/docs/ai/capabilities/process-history/), [Process Event Stream](/docs/ai/capabilities/process-event-stream/), [Reinject System Prompt](/docs/ai/capabilities/reinject-system-prompt/), and [Raise Content Filter Error](/docs/ai/capabilities/raise-content-filter-error/).

And the agent plugs into any interface: [ACP](/docs/ai/harness/acp/) *(experimental, Harness)* serves it to editors like Zed over the [Agent Client Protocol](https://agentclientprotocol.com), and core ships the [web chat UI](/docs/ai/guides/web/), [CLI](/docs/ai/integrations/cli/), [frontend adapters](/docs/ai/integrations/ui/overview/) (AG-UI, Vercel AI), and [realtime voice](/docs/ai/realtime/overview/).

Community packages extend the same capability system further; see [third-party capabilities](/docs/ai/capabilities/third-party/).

“Harness” is the field’s term for everything around the model that turns it into an agent: the loop, the tools, the context management. Reach for this package when your agent should *do* more than core’s lean harness covers: touch files, run code, browse, remember, delegate, or stay coherent through hours-long runs. The boundary between the packages is mechanical, not a maturity tier: core ships the capabilities that require model or framework support (provider-native tools like [image generation](/docs/ai/capabilities/image-generation/), provider APIs like [compaction](/docs/ai/capabilities/compaction/), deep loop integration like [tool search](/docs/ai/capabilities/tool-search/), and fundamentals like [thinking](/docs/ai/capabilities/thinking/), [MCP](/docs/ai/capabilities/mcp/), and [web search](/docs/ai/capabilities/web-search/)) and the Harness ships everything else, as a separate package so capabilities can iterate at the speed the field moves while Pydantic AI itself stays lean.

This installs [`pydantic-ai-slim`](/docs/ai/overview/install/) with it, so it works on its own; you don’t need to install Pydantic AI separately. Model providers and the CLI come via extras that pass through to Pydantic AI: `pydantic-ai-harness[anthropic]`, `[cli]`. Some capabilities need their own extra for optional dependencies; each capability’s page gives its exact install line. Requires Python 3.10+.

New to Pydantic AI itself? Start with [its docs](/docs/ai/): the agent you mount these capabilities on is defined there.

Everything the harness does is observable: core’s [Instrumentation](/docs/ai/capabilities/instrumentation/) capability (or `logfire.instrument_pydantic_ai()`) emits a full trace of every run: every model call and tool call, with token and cost tracking. It’s standard OpenTelemetry, so any OTLP backend works; [Logfire](https://pydantic.dev/logfire) is the easiest way to see it during development.

[Capabilities](/docs/ai/capabilities/custom/) are the primary extension point for Pydantic AI, and every capability in this library doubles as a worked example. Publishing a standalone package? Use the `pydantic-ai-<name>` naming convention; see [Publishing capability packages](/docs/ai/guides/extensibility/#publishing-capability-packages).

Pydantic AI Harness uses **0.x versioning**, and that’s a statement about API stability, not maturity: these capabilities are tested end-to-end and meant for production use, but their APIs may still move between minor releases (0.1 -> 0.2): renamed parameters, changed defaults, restructured APIs, always with deprecation warnings where practical. Patch releases will not intentionally break existing behavior, and every breaking change is documented in release notes with migration guidance your agent can follow. Keeping the Harness a separate package from [Pydantic AI](https://github.com/pydantic/pydantic-ai), which has a [stricter version policy](/docs/ai/project/version-policy/), is what lets capabilities iterate at the speed the field moves.

- [Capabilities](/docs/ai/capabilities/overview/) : what capabilities are, built-in capabilities, building your own
- [Hooks](/docs/ai/core-concepts/hooks/) : lifecycle hooks reference, ordering, error handling
- [Extensibility](/docs/ai/guides/extensibility/) : publishing packages, third-party ecosystem
- [Toolsets](/docs/ai/tools-toolsets/toolsets/) : building tools for capabilities
- [API reference](/docs/ai/api/pydantic-ai/capabilities/) : full API docs

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness
