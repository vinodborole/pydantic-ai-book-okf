---
type: Web Page
title: Capabilities | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/overview
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Capabilities

A capability is a reusable, composable unit of agent behavior. Instead of threading multiple arguments through your `Agent` constructor — [instructions](/docs/ai/core-concepts/agent/#instructions) here, [model settings](/docs/ai/core-concepts/agent/#model-run-settings) there, a [toolset](/docs/ai/tools-toolsets/toolsets/) somewhere else, a [history processor](/docs/ai/core-concepts/message-history/#processing-message-history) on yet another parameter — you can bundle related behavior into a single capability and pass it via the [`capabilities`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) parameter.

Capabilities can provide any combination of:

- **Tools** — via[toolsets](/docs/ai/tools-toolsets/toolsets/) or[native tools](/docs/ai/tools-toolsets/native-tools/)
- **Lifecycle hooks** — intercept and modify model requests, tool calls, and the overall run
- **Instructions** — static or dynamic[instruction](/docs/ai/core-concepts/agent/#instructions) additions
- **Model settings** — static or per-step[model settings](/docs/ai/core-concepts/agent/#model-run-settings)
- **Models** — static or adaptive model selection and application-specific model ID resolution

This makes them the primary extension point for Pydantic AI. Whether you’re building a memory system, a guardrail, a cost tracker, or an approval workflow, a capability is the right abstraction.

Capabilities can be always-on or [loaded by the model on demand](/docs/ai/capabilities/on-demand/). The [capability index below](#available-capabilities) spans Pydantic AI itself and [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/), [third-party packages](/docs/ai/capabilities/third-party/) provide many more, and you can define your own, [declaratively](#bundling-behavior-with-capability) or by [subclassing](/docs/ai/capabilities/custom/). To run agents durably across failures, restarts, and long waits, see [Durable Execution](/docs/ai/capabilities/durable_execution/overview/).

Capabilities come from two packages, and they all compose, with each other and with your own. Core (`pydantic-ai`) ships the capabilities that require model or framework support: provider-native tools, provider APIs, and deep loop integration. **[Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/)**, the official capability library and harness for Pydantic AI, ships everything else, from single capabilities to [complete agents](https://pydantic.dev/docs/ai/harness/coder/). The **Package** column says which; every entry links to its documentation.

Complete agent stacks as regular combined capabilities: one import gives you a working agent, and you can take either apart into the blocks below.

| Harness | Package | What it provides | 
|---|---|---|
| [Coder](https://pydantic.dev/docs/ai/harness/coder/) | Harness | A complete coding-agent stack: files, shell, repo context, planning, a read-only explorer sub-agent, and context controls | 
| [Researcher](https://pydantic.dev/docs/ai/harness/researcher/) | Harness | A complete web-research stack: search, page fetching, a delegated sub-researcher, and bounded tool output | 

The workspace the agent acts in: the files it edits and the commands it runs, local or isolated.

| Capability | Package | What it does | 
|---|---|---|
| [FileSystem](https://pydantic.dev/docs/ai/harness/filesystem/) | Harness | Read, write, edit, search files under a root; path-traversal and symlink safe, secrets read-only | 
| [Shell](https://pydantic.dev/docs/ai/harness/shell/) | Harness | Command execution with allowlists, denylists, timeouts, and credential-stripping | 
| [Modal Sandbox](https://pydantic.dev/docs/ai/harness/modal-sandbox/) | Harness | Commands and files in an isolated [Modal](https://modal.com) cloud sandbox | 

Connections to systems outside the agent’s workspace, and abilities the provider executes natively.

| Capability | Package | What it does | 
|---|---|---|
| [MCP](/docs/ai/capabilities/mcp/) | Core | Connect any MCP server’s tools; local by default, provider-native connectors opt-in | 
| [Image Generation](/docs/ai/capabilities/image-generation/) | Core | Generate and edit images; provider-native where supported, sub-agent fallback elsewhere | 
| [Native Tool](/docs/ai/tools-toolsets/native-tools/) | Core | Register any provider-native tool with the agent | 
| [StackOne](https://pydantic.dev/docs/ai/harness/stackone/) | Harness | Act on linked SaaS accounts (HRIS, ATS, CRM, …) via [StackOne](https://www.stackone.com) | 
| [LocalStack](https://pydantic.dev/docs/ai/harness/localstack/) | Harness | An emulated AWS environment with AWS CLI tools | 
| [Macroscope](https://pydantic.dev/docs/ai/harness/macroscope/) | Harness | Run a local [Macroscope](https://docs.macroscope.com/cli) code review and hand the findings to the agent | 

Finding and reading things on the open web.

| Capability | Package | What it does | 
|---|---|---|
| [Web Search](/docs/ai/capabilities/web-search/) | Core | Provider-native search where available, local DuckDuckGo fallback everywhere | 
| [Web Fetch](/docs/ai/capabilities/web-fetch/) | Core | Fetch and read URLs, native or local | 
| [X Search](/docs/ai/capabilities/x-search/) | Core | Search X; native on xAI, subagent fallback elsewhere | 
| [Exa Search](https://pydantic.dev/docs/ai/harness/exa-search/) | Harness | Web research via [Exa](https://exa.ai) : excerpted search, full-page reads, opt-in cited deep search | 
| [Exa Agent](https://pydantic.dev/docs/ai/harness/exa-search/) | Harness | Delegate open-ended research to the Exa Agent API | 
| [Browser Use](https://pydantic.dev/docs/ai/harness/browser-use/) | Harness | Hand web tasks to an autonomous [browser-use](https://github.com/browser-use/browser-use) agent driving a real browser | 

How the agent thinks and divides the work.

| Capability | Package | What it does | 
|---|---|---|
| [Thinking](/docs/ai/capabilities/thinking/) | Core | Provider-adaptive extended thinking at configurable effort | 
| [Planning](https://pydantic.dev/docs/ai/harness/planning/) | Harness | Model-owned task plans with a cache-safe live reminder | 
| [Subagents](https://pydantic.dev/docs/ai/harness/subagents/) | Harness | Delegate self-contained tasks to named child agents | 
| [Dynamic Workflow](https://pydantic.dev/docs/ai/harness/dynamic-workflow/) | Harness | The model orchestrates sub-agents from one Python script: fan-out, chain, vote in a single tool call, with hard `max_agent_calls` budgets | 
| [Advisor](https://pydantic.dev/docs/ai/harness/advisor/) | Harness | Let an executor consult a stronger model mid-run | 

How the agent spends its context window: the difference between an agent that degrades over a long run and one that doesn’t, and between paying for tokens N times or once.

| Capability | Package | What it does | 
|---|---|---|
| [Code Mode](https://pydantic.dev/docs/ai/harness/code-mode/) | Harness | The model writes one Python script that calls many tools inside a [Monty](https://github.com/pydantic/monty) sandbox: one round-trip instead of N, and intermediate results never enter the context window | 
| [Tool Search](/docs/ai/capabilities/tool-search/) | Core | Load tool definitions on demand instead of carrying hundreds in every prompt | 
| [Compaction](/docs/ai/capabilities/compaction/) | Core | Provider-native compaction on OpenAI and Anthropic; the provider summarizes history server-side | 
| [Compaction](https://pydantic.dev/docs/ai/harness/compaction/) | Harness | Model-agnostic strategies: tool-result clearing, sliding-window trimming, LLM summarization, tiered; all window-relative, with live usage reporting | 
| [Tool Output Limits](https://pydantic.dev/docs/ai/harness/tool-output-limits/) | Harness | Truncate, spill to a queryable file, or summarize oversized tool returns at the source | 
| [Warn On Cache Busts](https://pydantic.dev/docs/ai/harness/warn-on-cache-busts/) | Harness | Detect prompt-cache prefix collapses between requests, from the provider’s own numbers | 

What the agent knows and remembers, loaded when relevant instead of carried in every prompt.

| Capability | Package | What it does | 
|---|---|---|
| [Memory](https://pydantic.dev/docs/ai/harness/memory/) | Harness | A persistent, namespaced notebook: bounded prompt injection, on-demand search; in-memory/file/Postgres stores | 
| [Conversation Search](https://pydantic.dev/docs/ai/harness/conversation-search/) | Harness | BM25 search over stored history, including turns compaction dropped | 
| [Skills](https://pydantic.dev/docs/ai/harness/skills/) | Harness | Load [Agent Skill](/docs/ai/capabilities/on-demand/) (`SKILL.md` ) instructions on demand | 
| [Repo Context](https://pydantic.dev/docs/ai/harness/repo-context/) | Harness | Start runs oriented: `AGENTS.md` /`CLAUDE.md` + repository structure | 
| [Pydantic AI Docs](https://pydantic.dev/docs/ai/harness/pydantic-ai-docs/) | Harness | On-demand Pydantic AI documentation lookup | 

Bounding what the agent may do, and keeping it on-instructions.

| Capability | Package | What it does | 
|---|---|---|
| [Guardrails](https://pydantic.dev/docs/ai/harness/guardrails/) | Harness | Validate/block/redact user input, tool calls, tool results, and output, including secret masking and parallel async guards | 
| [Spend Limits](https://pydantic.dev/docs/ai/harness/spend/) | Harness | Cross-window USD/token budgets and per-response cost tracking, per model and per tenant | 
| [Tool approval](/docs/ai/tools-toolsets/deferred-tools/#human-in-the-loop-tool-approval) | Core | Flag tool calls that need human approval before they run | 
| [Handle Deferred Tool Calls](/docs/ai/capabilities/handle-deferred-tool-calls/) | Core | Resolve approval-deferred tool calls programmatically | 
| [System Reminders](https://pydantic.dev/docs/ai/harness/system-reminders/) | Harness | Cache-safe re-injection of guidance mid-run to counter instruction fade | 

| Capability | Package | What it does | 
|---|---|---|
| [Capability Creation](https://pydantic.dev/docs/ai/harness/capability-creation/) | Harness | The agent writes, validates, and persists *new capabilities* during a run, loaded on the next run: self-extension with typed, inspectable units instead of arbitrary code | 

Outside the loop: how runs persist, survive failures, and get observed and configured in production.

| Capability | Package | What it does | 
|---|---|---|
| [Durable execution](/docs/ai/capabilities/durable_execution/overview/) | Core | Runs that survive restarts and failures on [Temporal](/docs/ai/capabilities/durable_execution/temporal/) ,[DBOS](/docs/ai/capabilities/durable_execution/dbos/) , or[Prefect](/docs/ai/capabilities/durable_execution/prefect/) , with[Restate](/docs/ai/capabilities/durable_execution/restate/) ,[Kitaru](/docs/ai/capabilities/durable_execution/kitaru/) , and[Airflow](/docs/ai/capabilities/durable_execution/airflow/) integrations | 
| [Step Persistence](https://pydantic.dev/docs/ai/harness/step-persistence/) | Harness | Save, restore, resume ( `continue_run` ), and fork (`fork_run` ) runs; file/SQLite/Mongo backends | 
| [Instrumentation](/docs/ai/capabilities/instrumentation/) | Core | OpenTelemetry GenAI spans for every model and tool call; the raw material for [Logfire](https://pydantic.dev/logfire) traces | 
| [Managed Prompt](https://pydantic.dev/docs/ai/harness/managed-prompt/) | Harness | Back instructions with a [Logfire](https://pydantic.dev/logfire) -managed prompt; version and roll out without redeploying | 
| [Thread Executor](/docs/ai/capabilities/thread-executor/) | Core | Run sync tools on a shared thread pool | 

Core also ships capabilities for customizing the agent loop itself, mostly for production servers:

| Capability | Package | What it does | 
|---|---|---|
| [Hooks](/docs/ai/core-concepts/hooks/) | Core | Decorator-based lifecycle hook registration | 
| [Select Model](/docs/ai/capabilities/select-model/) | Core | Select a static or per-step model with a callable | 
| [Resolve Model ID](/docs/ai/capabilities/resolve-model-id/) | Core | Resolve custom, application-specific model IDs with a callable | 
| [Prepare Tools / Prepare Output Tools](/docs/ai/capabilities/prepare-tools/) | Core | Filter or modify function and [output tool](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput) definitions per step | 
| [Prefix Tools](/docs/ai/capabilities/prefix-tools/) | Core | Wrap a capability and prefix its tool names | 
| [Include Tool Return Schemas](/docs/ai/capabilities/include-tool-return-schemas/) | Core | Include return type schemas in tool definitions sent to the model | 
| [Set Tool Metadata](/docs/ai/capabilities/set-tool-metadata/) | Core | Merge metadata key-value pairs onto selected tools | 
| [Raise Content Filter Error](/docs/ai/capabilities/raise-content-filter-error/) | Core | Raise [`ContentFilterError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ContentFilterError) whenever a model response has`finish_reason='content_filter'` | 
| [Reinject System Prompt](/docs/ai/capabilities/reinject-system-prompt/) | Core | Reinject the configured system prompt when the incoming message history is missing one | 
| [Process History](/docs/ai/capabilities/process-history/) | Core | Wrap a [history processor](/docs/ai/core-concepts/message-history/#processing-message-history) | 
| [Process Event Stream](/docs/ai/capabilities/process-event-stream/) | Core | Forward [agent stream events](/docs/ai/capabilities/process-event-stream/) to a handler function | 

The authoring primitives, [`Capability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) for [bundling behavior without subclassing](#bundling-behavior-with-capability) and [`Toolset`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Toolset) for wrapping an [`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset), are covered below. [ACP](https://pydantic.dev/docs/ai/harness/acp/) *(experimental, Harness)* serves any agent to editors like Zed over the [Agent Client Protocol](https://agentclientprotocol.com). Capabilities that can be declared in [YAML/JSON agent specs](/docs/ai/core-concepts/agent-spec/#capability-spec-syntax) are listed there.

[Instructions](/docs/ai/core-concepts/agent/#instructions) and [model settings](/docs/ai/core-concepts/agent/#model-run-settings) are configured directly via the `instructions` and `model_settings` parameters on `Agent` (or [`AgentSpec`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentSpec)). Capabilities are for behavior that goes beyond simple configuration — tools, lifecycle hooks, and custom extensions. They compose well, especially when you want to reuse the same configuration across multiple agents or load it from a [spec file](/docs/ai/core-concepts/agent-spec/).

You don’t need a subclass to define a capability of your own: [`Capability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) bundles instructions, function tools, and [toolsets](/docs/ai/tools-toolsets/toolsets/) declaratively — think of it as defining a skill:

Add `defer_loading=True` and the bundle becomes an [on-demand capability](/docs/ai/capabilities/on-demand/) that stays collapsed to a one-line catalog entry until the model loads it — the same shape as [Agent Skills](/docs/ai/capabilities/on-demand/#loading-skills-from-markdown-files), which you can wrap in a `Capability` directly. See [The `Capability` convenience class](/docs/ai/capabilities/on-demand/#the-capability-convenience-class) for the full API. For behavior beyond instructions, tools, and toolsets — lifecycle hooks, model settings, native tools — subclass [`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) as covered in [Building Custom Capabilities](/docs/ai/capabilities/custom/).

[`WebSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch), [`WebFetch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch), [`ImageGeneration`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration), [`XSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.XSearch), and [`MCP`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.MCP) each cover a single capability (web search, URL fetch, image generation, X search, MCP) across two implementations:

- **Native** — invoked by the model provider when the model supports it. The work happens on the provider’s side (e.g. Anthropic’s web search runs server-side, returning results inline).
- **Local** — runs in your Python process. Used when the model doesn’t support the native tool; your code does the work (e.g. calling DuckDuckGo directly).

| Capability | Local fallback | Notes | 
|---|---|---|
| [`WebSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch) | `local='duckduckgo'` or`local=True` (DuckDuckGo) | Requires the `duckduckgo` optional group | 
| [`WebFetch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch) | `local=True` (markdownify-based fetch) | Requires the `web-fetch` optional group | 
| [`ImageGeneration`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration) | Subagent via `fallback_model=` | Delegates to a model that supports native image generation | 
| [`XSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.XSearch) | Subagent via `fallback_model=` | No default non-xAI fallback; set `fallback_model` to an xAI model that supports[`XSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool) | 
| [`MCP`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.MCP) | Direct connection to the MCP server (the default) | Accepts any [`MCPToolset`](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) input; transport is auto-detected from a URL | 

Because these capabilities contribute model-facing tools, their `id`, `description`, and `defer_loading` fields are meaningful: set them when that tool should stay hidden until the model loads the matching workflow with the `load_capability` tool. This includes [`ImageGeneration`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration) when image generation should only be available for an image-specific workflow, whether it resolves to a native image tool or a fallback subagent tool.

Configure each side via the `native=` and `local=` kwargs. `native=` accepts `True` (use the capability’s default [native tool](/docs/ai/tools-toolsets/native-tools/) instance), `False` (disable native), or an explicit instance like `WebSearchTool(...)` for fine-grained config. `local=` accepts `True` (the bundled local fallback, on capabilities that have one — `WebSearch` and `WebFetch`), `False` (disable local), a named strategy string where supported, or any callable, [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), or [`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset). Optional installs needed for the local fallback are opt-in — the capability raises a [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) at construction (with an install hint) when you ask for a local strategy whose extra isn’t installed.

`MCP` defaults the other way from the others: because MCP carries credentials, it runs locally by default and you opt into native MCP with `native=True`. The others default to native and you opt into local with `local=`.

[`XSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.XSearch) is slightly different from [`WebSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch) and [`WebFetch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch): there is no default non-xAI fallback. If your agent is not running on an xAI model, set `fallback_model` explicitly to an xAI model that supports [`XSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool).

Some constraint fields require the native tool (the bundled local fallback can’t enforce them) — passing them locks the capability to the native path. If the model doesn’t support the native tool, the capability raises a [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError).

All five capabilities are subclasses of [`NativeOrLocalTool`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.NativeOrLocalTool), which you can use directly or subclass to build your own provider-adaptive tools. For example, to pair [`CodeExecutionTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.CodeExecutionTool) with a local fallback:

Third-party packages publish capabilities of their own — see [Third-Party Capabilities](/docs/ai/capabilities/third-party/) for the ecosystem, and [Publishing capabilities](/docs/ai/capabilities/custom/#publishing-capabilities) for making your own capability available to others.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/overview
