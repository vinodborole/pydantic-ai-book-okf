---
type: Web Page
title: Overview | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/overview
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Overview

A capability is a reusable, composable unit of agent behavior. Instead of threading multiple arguments through your `Agent` constructor — [instructions](/docs/ai/core-concepts/agent#instructions) here, [model settings](/docs/ai/core-concepts/agent#model-run-settings) there, a [toolset](/docs/ai/tools-toolsets/toolsets) somewhere else, a [history processor](/docs/ai/core-concepts/message-history#processing-message-history) on yet another parameter — you can bundle related behavior into a single capability and pass it via the [ capabilities](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) parameter.

Capabilities can provide any combination of:

- **Tools**— via- [toolsets](/docs/ai/tools-toolsets/toolsets)or- [native tools](/docs/ai/tools-toolsets/native-tools)
- **Lifecycle hooks**— intercept and modify model requests, tool calls, and the overall run
- **Instructions**— static or dynamic- [instruction](/docs/ai/core-concepts/agent#instructions)additions
- **Model settings**— static or per-step- [model settings](/docs/ai/core-concepts/agent#model-run-settings)
- **Models**— static or adaptive model selection and application-specific model ID resolution

This makes them the primary extension point for Pydantic AI. Whether you’re building a memory system, a guardrail, a cost tracker, or an approval workflow, a capability is the right abstraction.

Capabilities can be always-on or [loaded by the model on demand](/docs/ai/capabilities/on-demand). Pydantic AI ships the built-in capabilities below, [Pydantic AI Harness](#pydantic-ai-harness) and [third-party packages](/docs/ai/capabilities/third-party) provide many more, and you can define your own — [declaratively](#bundling-behavior-with-capability) or by [subclassing](/docs/ai/capabilities/custom). To run agents durably across failures, restarts, and long waits, see [Durable Execution](/docs/ai/capabilities/durable_execution/overview).

Pydantic AI ships with several capabilities that cover common needs:

| Capability | What it provides | Spec | 
|---|---|---|
| `Thinking` | Enables model [thinking/reasoning](/docs/ai/capabilities/thinking)at configurable effort | Yes | 
| `Hooks` | Decorator-based [lifecycle hook](/docs/ai/core-concepts/hooks)registration | — | 
| `Instrumentation` | OpenTelemetry/Logfire [tracing](/docs/ai/capabilities/instrumentation)of runs, model requests, and tool calls | Yes | 
| `SelectModel` | Selects a static or per-step [model](/docs/ai/capabilities/select-model)with a callable | — | 
| `ResolveModelId` | Resolves custom [model IDs](/docs/ai/capabilities/resolve-model-id)with a callable | — | 
| `WebSearch` | [Web search](/docs/ai/capabilities/web-search)— native by default, optional[local fallback](/docs/ai/tools-toolsets/common-tools#duckduckgo-search-tool)via`local='duckduckgo'` | Yes | 
| `WebFetch` | [URL fetching](/docs/ai/capabilities/web-fetch)— native by default, optional[local fallback](/docs/ai/tools-toolsets/common-tools#web-fetch-tool)via`local=True` | Yes | 
| `ImageGeneration` | [Image generation](/docs/ai/capabilities/image-generation)— native by default, optional subagent fallback via`fallback_model` | Yes | 
| `XSearch` | [X search](/docs/ai/capabilities/x-search)— native on xAI, explicit subagent fallback via`fallback_model` | Yes | 
| `MCP` | [MCP server](/docs/ai/capabilities/mcp)— runs locally by default;`native=True`opts into the model provider’s native MCP support | Yes | 
| `ToolSearch` | [Discovery](/docs/ai/capabilities/tool-search)of[deferred tools](/docs/ai/tools-toolsets/tools-advanced#tool-search)— native when supported, local`search_tools`function tool otherwise | Yes | 
| `PrepareTools` | Filters or modifies function [tool definitions](/docs/ai/capabilities/prepare-tools)per step | — | 
| `PrepareOutputTools` | Filters or modifies [output tool](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput)[definitions](/docs/ai/capabilities/prepare-tools)per step | — | 
| `PrefixTools` | Wraps a capability and [prefixes its tool names](/docs/ai/capabilities/prefix-tools) | Yes | 
| `NativeTool` | Registers a [native tool](/docs/ai/tools-toolsets/native-tools)with the agent | Yes | 
| `Capability` | Bundles instructions, function tools, and toolsets [without subclassing](/docs/ai/capabilities/on-demand#the-capability-convenience-class) | — | 
| `Toolset` | Wraps an `AbstractToolset` | — | 
| `IncludeToolReturnSchemas` | [Includes return type schemas](/docs/ai/capabilities/include-tool-return-schemas)in tool definitions sent to the model | Yes | 
| `SetToolMetadata` | [Merges metadata key-value pairs](/docs/ai/capabilities/set-tool-metadata)onto selected tools | Yes | 
| `RaiseContentFilterError` | [Raises](/docs/ai/capabilities/raise-content-filter-error)whenever a model response has`ContentFilterError``finish_reason='content_filter'` | Yes | 
| `ReinjectSystemPrompt` | [Reinjects the configured system prompt](/docs/ai/capabilities/reinject-system-prompt)when the incoming message history is missing one | Yes | 
| `HandleDeferredToolCalls` | Resolves [deferred tool calls](/docs/ai/capabilities/handle-deferred-tool-calls)inline with a handler function | — | 
| `ProcessHistory` | Wraps a [history processor](/docs/ai/capabilities/process-history) | — | 
| `ProcessEventStream` | Forwards [agent stream events](/docs/ai/capabilities/process-event-stream)to a handler function | — | 
| `ThreadExecutor` | Uses a [custom thread executor](/docs/ai/capabilities/thread-executor)for[sync functions](/docs/ai/tools-toolsets/tools-advanced#thread-executor-for-long-running-servers) | — | 

The **Spec** column indicates whether the capability can be used in [agent specs](/docs/ai/core-concepts/agent-spec) (YAML/JSON). Capabilities marked **—** take non-serializable arguments (callables, toolset objects) and can only be used in Python code.

[Compaction](/docs/ai/capabilities/compaction) keeps conversations within the context window through several approaches; the provider-native [ OpenAICompaction](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAICompaction) and 

[capabilities live in the corresponding model modules. The](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicCompaction)

`AnthropicCompaction`[durable execution](/docs/ai/capabilities/durable_execution/overview)integrations also ship as capabilities —

[,](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability)

`TemporalDurability`[, and](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.dbos.DBOSDurability)

`DBOSDurability`[— in the](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.prefect.PrefectDurability)

`PrefectDurability``pydantic_ai.durable_exec` subpackages.[Instructions](/docs/ai/core-concepts/agent#instructions) and [model settings](/docs/ai/core-concepts/agent#model-run-settings) are configured directly via the `instructions` and `model_settings` parameters on `Agent` (or [ AgentSpec](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentSpec)). Capabilities are for behavior that goes beyond simple configuration — tools, lifecycle hooks, and custom extensions. They compose well, especially when you want to reuse the same configuration across multiple agents or load it from a 

[spec file](/docs/ai/core-concepts/agent-spec).

You don’t need a subclass to define a capability of your own: [ Capability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) bundles instructions, function tools, and 

[toolsets](/docs/ai/tools-toolsets/toolsets)declaratively — think of it as defining a skill:

Add `defer_loading=True` and the bundle becomes an [on-demand capability](/docs/ai/capabilities/on-demand) that stays collapsed to a one-line catalog entry until the model loads it — the same shape as [Agent Skills](/docs/ai/capabilities/on-demand#loading-skills-from-markdown-files), which you can wrap in a `Capability` directly. See [The  Capability convenience class](/docs/ai/capabilities/on-demand#the-capability-convenience-class) for the full API. For behavior beyond instructions, tools, and toolsets — lifecycle hooks, model settings, native tools — subclass 

[as covered in](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability)

`AbstractCapability`[Building Custom Capabilities](/docs/ai/capabilities/custom).

[ WebSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch), 

[,](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch)

`WebFetch`[,](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration)

`ImageGeneration`[, and](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.XSearch)

`XSearch`[each cover a single capability (web search, URL fetch, image generation, X search, MCP) across two implementations:](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.MCP)

`MCP`- **Native**— invoked by the model provider when the model supports it. The work happens on the provider’s side (e.g. Anthropic’s web search runs server-side, returning results inline).
- **Local**— runs in your Python process. Used when the model doesn’t support the native tool; your code does the work (e.g. calling DuckDuckGo directly).

| Capability | Local fallback | Notes | 
|---|---|---|
| `WebSearch` | `local='duckduckgo'`or`local=True`(DuckDuckGo) | Requires the `duckduckgo`optional group | 
| `WebFetch` | `local=True`(markdownify-based fetch) | Requires the `web-fetch`optional group | 
| `ImageGeneration` | Subagent via `fallback_model=` | Delegates to a model that supports native image generation | 
| `XSearch` | Subagent via `fallback_model=` | No default non-xAI fallback; set `fallback_model`to an xAI model that supports`XSearchTool` | 
| `MCP` | Direct connection to the MCP server (the default) | Accepts any input; transport is auto-detected from a URL`MCPToolset` | 

Because these capabilities contribute model-facing tools, their `id`, `description`, and `defer_loading` fields are meaningful: set them when that tool should stay hidden until the model loads the matching workflow with the `load_capability` tool. This includes [ ImageGeneration](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration) when image generation should only be available for an image-specific workflow, whether it resolves to a native image tool or a fallback subagent tool.

Configure each side via the `native=` and `local=` kwargs. `native=` accepts `True` (use the capability’s default [native tool](/docs/ai/tools-toolsets/native-tools) instance), `False` (disable native), or an explicit instance like `WebSearchTool(...)` for fine-grained config. `local=` accepts `True` (the bundled local fallback, on capabilities that have one — `WebSearch` and `WebFetch`), `False` (disable local), a named strategy string where supported, or any callable, [ Tool](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), or 

[. Optional installs needed for the local fallback are opt-in — the capability raises a](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset)

`AbstractToolset`[at construction (with an install hint) when you ask for a local strategy whose extra isn’t installed.](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError)

`UserError``MCP` defaults the other way from the others: because MCP carries credentials, it runs locally by default and you opt into native MCP with `native=True`. The others default to native and you opt into local with `local=`.

[ XSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.XSearch) is slightly different from 

[and](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch)

`WebSearch`[: there is no default non-xAI fallback. If your agent is not running on an xAI model, set](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch)

`WebFetch``fallback_model` explicitly to an xAI model that supports [.](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool)

`XSearchTool`Some constraint fields require the native tool (the bundled local fallback can’t enforce them) — passing them locks the capability to the native path. If the model doesn’t support the native tool, the capability raises a [ UserError](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError).

All five capabilities are subclasses of [ NativeOrLocalTool](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.NativeOrLocalTool), which you can use directly or subclass to build your own provider-adaptive tools. For example, to pair 

[with a local fallback:](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.CodeExecutionTool)

`CodeExecutionTool`[ Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) is the official capability library for Pydantic AI — standalone capabilities like memory, guardrails, context management, and 

[code mode](https://github.com/pydantic/pydantic-ai-harness/tree/main/pydantic_ai_harness/code_mode)live there rather than in core. See

[What goes where?](https://pydantic.dev/docs/ai/harness/#what-goes-where)for the full breakdown, or jump to the

[capability matrix](https://github.com/pydantic/pydantic-ai-harness#capability-matrix).

Third-party packages publish capabilities of their own — see [Third-Party Capabilities](/docs/ai/capabilities/third-party) for the ecosystem, and [Publishing capabilities](/docs/ai/capabilities/custom#publishing-capabilities) for making your own capability available to others.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/overview
