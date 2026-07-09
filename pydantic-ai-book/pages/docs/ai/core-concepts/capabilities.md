---
type: Web Page
title: Capabilities | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/capabilities
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Capabilities

A capability is a reusable, composable unit of agent behavior. Instead of threading multiple arguments through your `Agent` constructor — [instructions](/docs/ai/core-concepts/agent#instructions) here, [model settings](/docs/ai/core-concepts/agent#model-run-settings) there, a [toolset](/docs/ai/tools-toolsets/toolsets) somewhere else, a [history processor](/docs/ai/core-concepts/message-history#processing-message-history) on yet another parameter — you can bundle related behavior into a single capability and pass it via the [ capabilities](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) parameter.

Capabilities can provide any combination of:

- **Tools**— via- [toolsets](/docs/ai/tools-toolsets/toolsets)or- [native tools](/docs/ai/overview/native-tools)
- **Lifecycle hooks**— intercept and modify model requests, tool calls, and the overall run
- **Instructions**— static or dynamic- [instruction](/docs/ai/core-concepts/agent#instructions)additions
- **Model settings**— static or per-step- [model settings](/docs/ai/core-concepts/agent#model-run-settings)

This makes them the primary extension point for Pydantic AI. Whether you’re building a memory system, a guardrail, a cost tracker, or an approval workflow, a capability is the right abstraction.

A multi-workflow agent normally sends every workflow’s instructions and tool schemas on every turn, and applies every workflow’s settings and hooks for the whole run — even though most requests need just one workflow. That cost grows with each workflow you add: more input tokens, and worse tool selection once the visible tool set passes the ~30–50-tool mark where models start picking the wrong one (the same pressure behind [tool search](/docs/ai/tools-toolsets/tools-advanced#tool-search)).

Mark a capability with `defer_loading=True` and give it a stable `id`, and it collapses to a one-line catalog entry — its `id` plus an optional `description` — that the model pulls in on demand. Here’s the minimal shape:

On the first turn, the refund workflow is collapsed to a catalog entry. The model sees its base instructions, the framework-managed `load_capability` tool, and the catalog appended to the instructions:

```
The following capabilities are deferred and can be loaded using the `load_capability` tool:
- refunds: Use for refund eligibility, refund status, or processing a refund.
```
The model does not receive the refund instructions, and `refund_status` is not callable yet. Depending on the active model, Pydantic AI may also send provider/tool-search plumbing to preserve the hidden state; that plumbing does not expose the refund tool until the capability is loaded. The exchange unfolds across model requests within a single `agent.run_sync` call:

- **Request 1.**The model sees the catalog above and the user’s prompt. It calls the- `load_capability`tool with- `id='refunds'`.
- **Load.**Pydantic AI returns the capability’s instructions —- *“Always confirm the order ID before issuing a refund.”*— as the tool result, and registers- `refund_status`for the next request.
- **Request 2.**The model now sees those instructions in history and- `refund_status`in its tool list. It calls- `refund_status(order_id='ABC-123')`and answers the user from the result.

Already-loaded capabilities stay loaded for the rest of the run — the model never needs to re-open one.

Loading activates the whole bundle, not just instructions: the capability’s function tools, model settings, and lifecycle hooks come live together (see [What you can defer](#what-you-can-defer)). It’s a one-line change to a capability you already register, it works on [every provider](#cross-provider-behavior), and it [survives history replay](#resumable-across-runs).

Every part of a capability bundle activates together as a single unit:

| Part | Before load | After load | 
|---|---|---|
| Instructions (static or dynamic) | Not sent | Returned as the `load_capability`tool result; included in subsequent requests | 
| Function tools | Not exposed | Exposed on the next request | 
| Model settings (static or per-step) | Not applied | Merged into the run’s settings for subsequent requests | 
| Lifecycle [hooks](#hooking-into-the-lifecycle) | Do not fire | Fire after the capability is loaded | 
| [Native tools](/docs/ai/overview/native-tools) | Not exposed | Exposed on the next request — see [Cache implications](#cache-implications) | 

**Reach for on-demand capabilities when:**

- the agent serves multiple distinct workflows (refunds, returns, fraud review, account security…) where most turns need one
- a workflow needs *more than instructions*— its own tools, raised reasoning effort, an approval hook — and those should travel together as a unit
- you want skills-style progressive disclosure but also want the loaded bundle to bring tools and settings, not just a runbook

**Skip it when:**

- the capability is used on most turns — the discovery round-trip costs more than the tokens it saves
- you have a flat catalog of individually-discoverable tools with no shared instructions — use [tool search](/docs/ai/tools-toolsets/tools-advanced#tool-search)instead, which discovers individual tools by name rather than loading bundles

If you’ve used [Anthropic’s Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills), this is the same idea generalised: a skill is a markdown file the model can pull in on demand. An on-demand capability does that *plus* typed function tools, per-step model settings, and lifecycle hooks.

`defer_loading=True` is not specific to the [ Capability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) convenience class. The shared fields live on 

[, and built-in capabilities expose](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability)

`AbstractCapability``id`, `description`, and `defer_loading` on construction. For custom capabilities, set those attributes on the instance.Until the model loads `analytics-mcp`, none of the MCP server’s tool definitions enter the prompt. The same flag works on [ WebSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch), 

[,](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch)

`WebFetch`[, and any custom](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks)

`Hooks`[subclass — see](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability)

`AbstractCapability`[Building custom capabilities](#building-custom-capabilities)for adding

`defer_loading` to your own subclass.Loaded-capability state lives in message history, not in the agent. When a conversation is persisted to a database and resumed later — possibly on a different process, machine, or model — Pydantic AI reconstructs the loaded set from the `load_capability` tool call/return pairs in history. Capabilities the model loaded earlier stay loaded; capabilities it never loaded stay collapsed in the catalog. No re-discovery round-trip on resume.

This is why deferred capabilities require a stable explicit `id`: history replay matches calls to capabilities by id, so a class-derived id would silently break the moment a class is renamed. The same property makes cross-provider replay work — a run that loaded `refunds` on Anthropic and continued on OpenAI Responses keeps `refunds` loaded after the switch.

History carries *which* capability ids were loaded, not the capabilities themselves: the resuming agent must be constructed with the same capabilities (matching `id`s), just as it must be constructed with the same tools. State lives in history; definitions live in code.

Several [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) fields expose progressive-disclosure state to tools, hooks, and capability-owned callbacks:

- `ctx.loaded_capability_ids`— deferred capability IDs explicitly loaded through the- `load_capability`tool, reconstructed from message history and updated when a capability loads during the current step.
- `ctx.available_capability_ids`— the currently-live capability IDs: always-available capabilities plus- `ctx.loaded_capability_ids`.
- `ctx.capability_loaded`— only meaningful while Pydantic AI is running a capability-owned hook or callback. It is scoped to that capability; deferred hooks and callbacks are skipped until this value would be true.
- `ctx.discovered_tool_names`— deferred function tools revealed by tool search. This is tool-level discovery, separate from capability-level loading.
- `ctx.available_tool_names`— function tool names currently known as available: always-visible tools from the current step’s assembled tool manager plus tool-search discoveries reconstructed from history. Early hooks such as- `before_run`may see only the history-derived discovered names, or an empty set if none exist yet, before tool definitions have been prepared. See- [Hook ordering](/docs/ai/core-concepts/hooks#hook-ordering)for how hook timing affects what is populated.

Loading a capability updates the capability state immediately, but the loaded bundle’s function tools, native tools, and model settings take effect on the next model request.

On-demand capabilities work on every model. Where the provider exposes a native progressive-disclosure surface — Anthropic tool search on Sonnet 4.5+/Opus 4.5+/Haiku 4.5+, OpenAI Responses `tool_search` on GPT-5.4+ — Pydantic AI uses that surface so deferred function tools stay out of the prompt prefix. Standalone deferred tools can use the provider’s hosted search; tools owned by on-demand capabilities use client-executed local search through the native surface so tools from unloaded capabilities cannot leak. On other providers, a local `search_tools` function tool handles discovery: the initial context shrinks the same way, but cache stability across loads is not guaranteed.

Calling the `load_capability` tool reveals capability behavior between requests. Whether that breaks the provider’s prompt-cache prefix depends on what’s revealed:

| What loads | Cache prefix | 
|---|---|
| Instructions only | Stable— instructions land in the message history, not the request prefix. | 
| Function tools on a model with native [tool search](/docs/ai/tools-toolsets/tools-advanced#tool-search)(OpenAI Responses, Anthropic) | Stable— the function tools visible to the provider don’t change across loads. | 
| Function tools on other models (local `search_tools`fallback) | May break between turns— function-tool visibility changes as capabilities load. | 
| Native tools | Always breaks the prefix on load— native tool definitions are part of the request prefix on every provider. | 

When preserving the cache prefix matters, prefer instruction-only or function-tool-only on-demand capabilities on a model with native tool search. The provider-specific mechanics that keep the prefix stable live in [tools-advanced.md](/docs/ai/tools-toolsets/tools-advanced#tool-search).

[ Capability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) bundles instructions, function tools, and toolsets without subclassing. Register tools with the decorator that mirrors 

[:](/docs/ai/tools-toolsets/tools#registering-function-tools-via-decorator)

`@agent.tool`In addition to `@capability.tool` and `@capability.tool_plain`, you can pass existing functions or [ Tool](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) instances via 

`tools=`, or hand in one or more [toolsets](/docs/ai/tools-toolsets/toolsets)via

`toolsets=`. For dynamic instructions, use the [decorator. For a dynamic catalog entry, pass a callable as](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability.instructions)

`@capability.instructions``description=`.`@capability.tool` and `@capability.tool_plain` mirror [ @agent.tool](/docs/ai/tools-toolsets/tools#registering-function-tools-via-decorator) exactly, including the 

`defer_loading` argument. On a deferred capability that per-tool flag is a no-op — the capability gates all its tools as a unit — so it only has an effect on a non-deferred `Capability`, where it opts an individual tool into [tool search](/docs/ai/tools-toolsets/tools-advanced#tool-search)discovery.

For anything beyond instructions, function tools, toolsets, and descriptions — model settings, hooks, native tools, wrapper toolsets, or custom per-run logic — subclass [ AbstractCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) directly. When subclassing, override 

[if the catalog entry needs to vary by run.](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_description)

`get_description`The [ Capability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) example above deferred instructions and a function tool, but the same flag gates the whole bundle — what the model knows, what it can do, and how it does it (see 

[What you can defer](#what-you-can-defer)). The snippets below show the remaining pieces in turn: model settings, hooks, and native tools.

[ get_model_settings](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model_settings) is collected during capability assembly, but its settings are only applied after the deferred capability is loaded. That means per-step settings like raised reasoning effort only apply for workflows the model opts into:

Hooks can live on deferred capabilities too. They do not run until the model loads the capability that owns them:

Any [native capability](#native-capabilities) (`WebSearch`, `WebFetch`, `MCP`, …) can be deferred the same way. The native tool definition only enters the request after the `load_capability` tool loads the capability — see [Cache implications](#cache-implications) for the trade-off:

A realistic on-demand capability rarely consists of just one piece. The example below defines a customer-support agent with two deferred workflows that exercise different parts of the bundle:

- `orders`— instructions plus a function tool, defined inline with- `Capability`
- `account-security`— instructions, a function tool, raised reasoning effort,- *and*an approval hook, all bundled as one- `AbstractCapability`

For those workflows, turn 1 exposes only the two-line catalog. Base instructions, always-on tools, the framework-managed `load_capability` tool, and any provider/tool-search plumbing still appear as usual. Loading `account-security` activates the runbook, the destructive tool, the higher reasoning effort, *and* the approval gate together — that’s what we mean by bundle-level disclosure.

A “where is my order?” request loads only `orders`. A “someone is logging into my account” request loads only `account-security` — and from that point on, every tool call in the run passes through the approval hook *and* benefits from the raised reasoning effort, without either being visible to the model on requests that never touched the workflow.

Want the model to actually *read the runbook* before taking a destructive action? Make the runbook a deferred capability, then check `ctx.loaded_capability_ids` in a one-method hook:

The model sees `issue_refund` from turn 1. If it tries to call it before opening `refund-policy`, the hook bounces the call back with a message pointing at the exact `load_capability` tool call to make. The model loads the policy, the policy text lands in its recent context, and the refund runs *within* the rules — and only then. Same shape for any tool-and-runbook pair.

Because the loaded set is just runtime data on [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext), the pattern generalises: dynamic instructions can warn when a risky pair of workflows is open, audit hooks can tag traces with the loaded set, escalation hooks can require an extra confirmation when both 

`payments` and `account-security` are active.If you already keep your skills as Markdown files with YAML frontmatter — the format used by [Anthropic Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) — you can wrap each one in a [ Capability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) with a few lines of glue.

Given a skill file `skills/refunds.md`:

Load it into an agent as an on-demand capability:

Each file shows up in the model’s catalog as its `id` plus `description`; the body is only sent once the model calls the `load_capability` tool. To go beyond instructions — add function tools, model settings, or hooks for a particular skill — subclass [ AbstractCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) as in the examples above.

Pydantic AI ships with several capabilities that cover common needs:

| Capability | What it provides | Spec | 
|---|---|---|
| `Thinking` | Enables model [thinking/reasoning](/docs/ai/advanced-features/thinking)at configurable effort | Yes | 
| `Hooks` | Decorator-based [lifecycle hook](/docs/ai/core-concepts/hooks)registration | — | 
| `Instrumentation` | OpenTelemetry/Logfire tracing — see [Debugging and Monitoring](/docs/ai/integrations/logfire) | Yes | 
| `WebSearch` | Web search — native by default, optional [local fallback](/docs/ai/tools-toolsets/common-tools#duckduckgo-search-tool)via`local='duckduckgo'` | Yes | 
| `WebFetch` | URL fetching — native by default, optional [local fallback](/docs/ai/tools-toolsets/common-tools#web-fetch-tool)via`local=True` | Yes | 
| `ImageGeneration` | Image generation — native by default, optional subagent fallback via `fallback_model` | Yes | 
| `XSearch` | X search — native on xAI, explicit subagent fallback via `fallback_model` | Yes | 
| `MCP` | MCP server — runs locally by default; `native=True`opts into the model provider’s native MCP support | Yes | 
| `ToolSearch` | Discovery of [deferred tools](/docs/ai/tools-toolsets/tools-advanced#tool-search)— native when supported, local`search_tools`function tool otherwise | Yes | 
| `PrepareTools` | Filters or modifies function [tool definitions](/docs/ai/tools-toolsets/tools)per step | — | 
| `PrepareOutputTools` | Filters or modifies [output tool](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput)definitions per step | — | 
| `PrefixTools` | Wraps a capability and prefixes its tool names | Yes | 
| `NativeTool` | Registers a [native tool](/docs/ai/overview/native-tools)with the agent | Yes | 
| `Capability` | Bundles instructions, function tools, and toolsets without subclassing | — | 
| `Toolset` | Wraps an `AbstractToolset` | — | 
| `IncludeToolReturnSchemas` | Includes return type schemas in tool definitions sent to the model | Yes | 
| `SetToolMetadata` | Merges metadata key-value pairs onto selected tools | Yes | 
| `HandleDeferredToolCalls` | Resolves [deferred tool calls](/docs/ai/tools-toolsets/deferred-tools#resolving-deferred-calls-with-a-handler)inline with a handler function | — | 
| `ProcessHistory` | Wraps a [history processor](/docs/ai/core-concepts/message-history#processing-message-history) | — | 
| `ProcessEventStream` | Forwards agent stream events to a handler function | — | 
| `ThreadExecutor` | Uses a custom thread executor for [sync functions](/docs/ai/tools-toolsets/tools-advanced#thread-executor-for-long-running-servers) | — | 

The **Spec** column indicates whether the capability can be used in [agent specs](/docs/ai/core-concepts/agent-spec) (YAML/JSON). Capabilities marked **—** take non-serializable arguments (callables, toolset objects) and can only be used in Python code.

[Instructions](/docs/ai/core-concepts/agent#instructions) and [model settings](/docs/ai/core-concepts/agent#model-run-settings) are configured directly via the `instructions` and `model_settings` parameters on `Agent` (or `AgentSpec`). Capabilities are for behavior that goes beyond simple configuration — tools, lifecycle hooks, and custom extensions. They compose well, especially when you want to reuse the same configuration across multiple agents or load it from a [spec file](/docs/ai/core-concepts/agent-spec).

The [ Thinking](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Thinking) capability enables model 

[thinking/reasoning](/docs/ai/advanced-features/thinking)at a configurable effort level. It’s the simplest way to enable thinking across providers:

See [Thinking](/docs/ai/advanced-features/thinking) for provider-specific details and the [unified thinking settings](/docs/ai/advanced-features/thinking#unified-thinking-settings).

Provider-specific compaction capabilities manage conversation context size by compacting older messages into summaries:

| Provider | Capability | Details | 
|---|---|---|
| OpenAI Responses API | `OpenAICompaction` | [OpenAI compaction](/docs/ai/models/openai#message-compaction) | 
| Anthropic | `AnthropicCompaction` | [Anthropic compaction](/docs/ai/models/anthropic#message-compaction) | 

The [ ThreadExecutor](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ThreadExecutor) capability provides a custom 

[for running sync tool functions and other sync callbacks in threads. This is useful in long-running servers (e.g. FastAPI) where the default ephemeral threads from](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.Executor)

`Executor``anyio.to_thread.run_sync` can accumulate under sustained load:```
from concurrent.futures import ThreadPoolExecutor
from pydantic_ai import Agent
from pydantic_ai.capabilities import ThreadExecutor
executor = ThreadPoolExecutor(max_workers=16, thread_name_prefix='agent-worker')
agent = Agent('openai:gpt-5.2', capabilities=[ThreadExecutor(executor)])
```
See [Thread executor for long-running servers](/docs/ai/tools-toolsets/tools-advanced#thread-executor-for-long-running-servers) for more details.

The [ Hooks](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) capability provides decorator-based 

[lifecycle hook](/docs/ai/core-concepts/hooks)registration — the easiest way to intercept model requests, tool calls, and other events without subclassing

[:](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability)

`AbstractCapability````
from pydantic_ai import Agent, ModelRequestContext, RunContext
from pydantic_ai.capabilities import Hooks
hooks = Hooks()
@hooks.on.before_model_request
async def log_request(ctx: RunContext, request_context: ModelRequestContext) -> ModelRequestContext:
    agent_name = ctx.agent.name if ctx.agent else 'unknown'
    print(f'[{agent_name}] Sending {len(request_context.messages)} messages')
    return request_context
agent = Agent('openai:gpt-5.2', name='my_agent', capabilities=[hooks])
```
All hooks receive [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext), which provides access to the running agent via 

[— useful for logging, metrics, and other cross-cutting concerns that need to identify which agent is running.](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.agent)

`ctx.agent`Hooks can also push follow-up messages into the conversation via
[ RunContext.enqueue](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.enqueue) — useful for capability
authors that need to surface an event to the model mid-run without rebuilding the
cached system prompt. See 

[Injecting messages mid-run](/docs/ai/core-concepts/message-history#injecting-messages-mid-run).

See the dedicated [Hooks](/docs/ai/core-concepts/hooks) page for the full API: decorator and constructor registration, timeouts, tool filtering, wrap hooks, per-event hooks, and more.

[ WebSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch), 

[,](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch)

`WebFetch`[,](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration)

`ImageGeneration`[, and](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.XSearch)

`XSearch`[each cover a single capability (web search, URL fetch, image generation, X search, MCP) across two implementations:](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.MCP)

`MCP`- **Native**— invoked by the model provider when the model supports it. The work happens on the provider’s side (e.g. Anthropic’s web search runs server-side, returning results inline).
- **Local**— runs in your Python process. Used when the model doesn’t support the native tool; your code does the work (e.g. calling DuckDuckGo directly).

Because these capabilities contribute model-facing tools, their `id`, `description`, and `defer_loading` fields are meaningful: set them when that tool should stay hidden until the model loads the matching workflow with the `load_capability` tool. This includes [ ImageGeneration](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration) when image generation should only be available for an image-specific workflow, whether it resolves to a native image tool or a fallback subagent tool.

Configure each side via the `native=` and `local=` kwargs. `native=` accepts `True` (use the capability’s default [native tool](/docs/ai/overview/native-tools) instance), `False` (disable native), or an explicit instance like `WebSearchTool(...)` for fine-grained config. `local=` accepts `True` (the bundled local fallback), `False` (disable local), a named strategy string where supported, or any callable, [ Tool](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), or 

[. Optional installs needed for the local fallback are opt-in — the capability raises a](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset)

`AbstractToolset`[at construction (with an install hint) when you ask for a local strategy whose extra isn’t installed.](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError)

`UserError``MCP` defaults the other way from the others: because MCP carries credentials, it runs locally by default and you opt into native MCP with `native=True`. The others default to native and you opt into local with `local=`.

[ XSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.XSearch) is slightly different from 

[and](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch)

`WebSearch`[: there is no default non-xAI fallback. If your agent is not running on an xAI model, set](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch)

`WebFetch``fallback_model` explicitly to an xAI model that supports [.](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool)

`XSearchTool`Some constraint fields require the native tool (the bundled local fallback can’t enforce them) — passing them locks the capability to the native path. If the model doesn’t support the native tool, the capability raises a [ UserError](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError).

[ WebSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch) defaults to native-only. Backed by 

[on the native side (see](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebSearchTool)

`WebSearchTool`[Web Search Tool](/docs/ai/overview/native-tools#web-search-tool)for provider support and configuration) — pass

`native=WebSearchTool(...)` directly when you need full control over the native instance.For the local side, pass `local='duckduckgo'` (or `local=True`) for a [DuckDuckGo](/docs/ai/tools-toolsets/common-tools#duckduckgo-search-tool) fallback (requires the `duckduckgo` optional group); for other search providers, use a [Tavily](/docs/ai/api/pydantic-ai/common_tools/#pydantic_ai.common_tools.tavily.tavily_search_tool) or [Exa](/docs/ai/api/pydantic-ai/common_tools/#pydantic_ai.common_tools.exa.ExaSearchTool) wrapper from [ common_tools](/docs/ai/tools-toolsets/common-tools), or any callable, 

[, or](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool)

`Tool`[.](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset)

`AbstractToolset`Native constraint fields: `search_context_size`, `user_location`, `blocked_domains`, `allowed_domains`, `max_uses`. The domain and `max_uses` constraints require native support (the shipped DuckDuckGo fallback doesn’t enforce them).

[ WebFetch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch) defaults to native-only. Backed by 

[on the native side (see](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebFetchTool)

`WebFetchTool`[Web Fetch Tool](/docs/ai/overview/native-tools#web-fetch-tool)for provider support and configuration) — pass

`native=WebFetchTool(...)` directly for full control.For the local side, pass `local=True` for the bundled [markdownify-based fetch tool](/docs/ai/tools-toolsets/common-tools#web-fetch-tool) (requires the `web-fetch` optional group), or any callable, [ Tool](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), or 

[.](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset)

`AbstractToolset`Native constraint fields: `allowed_domains`, `blocked_domains`, `max_uses`, `enable_citations`, `max_content_tokens`. Only `max_uses` requires native; domain filters are enforced locally when native isn’t available.

[ ImageGeneration](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ImageGeneration) defaults to native-only. Backed by 

[on the native side (see](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.ImageGenerationTool)

`ImageGenerationTool`[Image Generation Tool](/docs/ai/overview/native-tools#image-generation-tool)for provider support and configuration) — pass

`native=ImageGenerationTool(...)` directly for full control.For the local side, pass `fallback_model='…'` to delegate unsupported requests to a subagent running an image-generation-capable model (e.g. `openai-responses:gpt-5.4`), or `local=` with any callable, [ Tool](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), or 

[for a custom generator.](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset)

`AbstractToolset`[ MCP](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.MCP) is the primary entry point for 

[MCP](/docs/ai/mcp/overview)in Pydantic AI. It runs the MCP server locally by default — keeping credentials, hooks, and tracing under your control — and supports both URL-based servers and direct client / toolset / transport inputs.

Backed by [ MCPServerTool](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.MCPServerTool) on the native side (see 

[MCP Server Tool](/docs/ai/overview/native-tools#mcp-server-tool)for provider support and configuration) — pass

`native=MCPServerTool(...)` directly when you need full control (e.g. a different `id`, `authorization_token`, or `description` than the capability would derive). On the local side, `local=` accepts any [input (URL,](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset)

`MCPToolset``fastmcp.Client`, transport, in-process `FastMCP` server, script path, …) — non-toolset inputs are wrapped in `MCPToolset` automatically.For lower-level access — managing the [ MCPToolset](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) lifecycle directly, advanced transport / client configuration, or using MCP servers without going through a capability — see the 

[MCP documentation](/docs/ai/mcp/overview).

All four capabilities are subclasses of [ NativeOrLocalTool](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.NativeOrLocalTool), which you can use directly or subclass to build your own provider-adaptive tools. For example, to pair 

[with a local fallback:](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.CodeExecutionTool)

`CodeExecutionTool`The [ ToolSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch) capability handles discovery of tools marked with 

`defer_loading=True`, so agents with large toolsets only pay tokens for the tools the model needs. Like the [provider-adaptive tools](#provider-adaptive-tools)above, it picks the best path for the active model — native server-executed search on Anthropic and OpenAI Responses, a local

`search_tools` function tool elsewhere — and is auto-injected into every agent with zero overhead when no deferred tools exist.Pass an explicit [ ToolSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch) to pick a specific 

[(](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch.strategy)

`strategy``'keywords'`, `'bm25'`, `'regex'`, or a custom callable) or tune the local fallback:See [Tool Search](/docs/ai/tools-toolsets/tools-advanced#tool-search) for when to reach for it, the full strategy table, and provider support details.

[ PrepareTools](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrepareTools) and 

[wrap a](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrepareOutputTools)

`PrepareOutputTools`[as a capability, for filtering or modifying](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolsPrepareFunc)

`ToolsPrepareFunc`[tool definitions](/docs/ai/tools-toolsets/tools)per step.

`PrepareTools` handles function tools; `PrepareOutputTools` handles [output tools](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput).

For more complex tool preparation logic, see [Tool preparation](#tool-preparation) under lifecycle hooks.

[ PrefixTools](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrefixTools) wraps another capability and prefixes all of its tool names, useful for namespacing when composing multiple capabilities that might have conflicting tool names:

Every [ AbstractCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) has a convenience method 

[that returns a](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.prefix_tools)

`prefix_tools`[wrapper:](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrefixTools)

`PrefixTools`[ IncludeToolReturnSchemas](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.IncludeToolReturnSchemas) includes return type schemas in tool definitions sent to the model. For models that natively support return schemas (e.g. Google Gemini), the schema is passed as a structured field in the API request. For other models, it is injected into the tool description as JSON text.

*(This example is complete, it can be run “as is”)*

Use the `tools` parameter to select which tools should include return schemas. It accepts a list of tool names, a metadata dict for matching, or a callable predicate:

*(This example is complete, it can be run “as is”)*

The same effect can be achieved at the toolset level using [ .include_return_schemas()](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.include_return_schemas) — see 

[toolset composition](/docs/ai/tools-toolsets/toolsets#including-return-schemas).

[ SetToolMetadata](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.SetToolMetadata) merges metadata key-value pairs onto selected tools. This is useful for tagging tools with configuration that other capabilities or custom logic can inspect:

*(This example is complete, it can be run “as is”)*

The same effect can be achieved at the toolset level using [ .with_metadata()](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.with_metadata) — see 

[toolset composition](/docs/ai/tools-toolsets/toolsets#setting-tool-metadata).

[ ReinjectSystemPrompt](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ReinjectSystemPrompt) ensures the agent’s configured 

[is at the head of the first](/docs/ai/core-concepts/agent#system-prompts)

`system_prompt`[on every model request. By default, if any](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelRequest)

`ModelRequest`[is already present in the history, the capability is a no-op (so multi-agent handoff and user-managed system prompts remain authoritative). Set](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.SystemPromptPart)

`SystemPromptPart``replace_existing=True` to instead strip any existing `SystemPromptPart`s before prepending the agent’s configured prompt — useful when the history comes from an untrusted source and the server’s prompt must win.Useful when `message_history` comes from a source that doesn’t round-trip system prompts — UI frontends, database persistence layers, conversation compaction pipelines. Without this capability, an agent configured with a `system_prompt` will silently run without it if the history doesn’t already include one.

*(This example is complete, it can be run “as is”)*

The [UI adapters](/docs/ai/integrations/ui/ag-ui) (AG-UI, Vercel AI) automatically add this capability with `replace_existing=True` in their `manage_system_prompt='server'` mode.

To build your own capability, subclass [ AbstractCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) and override the methods you need. There are two categories: 

**configuration methods**that are called at agent construction (except

[which is called per-run), and](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_wrapper_toolset)

`get_wrapper_toolset`**lifecycle hooks**that fire during each run.

Custom capability classes can be plain classes or dataclasses. The shared metadata attributes — [ id](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.id), 

[, and](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.description)

`description`[— are optional declarations on the capability object for always-available capabilities. If](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.defer_loading)

`defer_loading``id` is omitted there, Pydantic AI derives a run-local id from the class name and disambiguates duplicates within the run. Deferred capabilities require an explicit stable `id`.Use a dataclass when you want generated constructor parameters for your own configuration fields, or for the shared metadata fields:

If you define a custom `__init__`, set only the metadata you want to expose. There is no `super().__init__()` or `__post_init__()` requirement:

When [ defer_loading=True](#on-demand-capabilities), provide a stable explicit 

`id`; history replay depends on it, and Pydantic AI rejects deferred capabilities without one. For always-available capabilities, omitting `id` still derives a run-local id from the class name.A capability that provides tools returns a [toolset](/docs/ai/tools-toolsets/toolsets) from [ get_toolset](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_toolset). This can be a pre-built 

[instance, or a callable that receives](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset)

`AbstractToolset`[and returns one dynamically:](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext`For [native tools](/docs/ai/overview/native-tools), override [ get_native_tools](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_native_tools) to return a sequence of 

[instances (which includes both](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.AgentNativeTool)

`AgentNativeTool`[objects and callables that receive](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.AbstractNativeTool)

`AbstractNativeTool`[).](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext`[ get_wrapper_toolset](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_wrapper_toolset) lets a capability wrap the agent’s entire assembled toolset with a 

[. This is more powerful than providing tools — it can intercept tool execution, add logging, or apply cross-cutting behavior.](/docs/ai/tools-toolsets/toolsets#changing-tool-execution)

`WrapperToolset`The wrapper receives the combined non-output toolset (after the [ prepare_tools](#tool-preparation) hook has wrapped it). Output tools are added separately and are not affected.

[ get_instructions](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_instructions) adds 

[instructions](/docs/ai/core-concepts/agent#instructions)to the agent. Since it’s called once at agent construction, return a callable if you need dynamic values:

Instructions can also use [template strings](/docs/ai/core-concepts/agent-spec#template-strings) (`TemplateStr('Hello {{name}}')`) for Handlebars-style templates rendered against the agent’s [dependencies](/docs/ai/core-concepts/dependencies). In Python code, a callable with [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) is generally preferred for IDE autocomplete.

[ get_model_settings](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model_settings) returns 

[model settings](/docs/ai/core-concepts/agent#model-run-settings)as a dict or a callable for per-step settings.

When model settings need to vary per step — for example, enabling thinking only on retry, or forcing a specific [ tool_choice](/docs/ai/tools-toolsets/tools-advanced#dynamic-tool-choice-via-capabilities) until a tool has been called — return a callable:

The callable receives a [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) where 

`ctx.model_settings` contains the merged result of all layers resolved before this capability (model defaults and agent-level settings).| Method | Return type | Purpose | 
|---|---|---|
| `get_toolset()` | `AgentToolset`` | None` | A [toolset](/docs/ai/tools-toolsets/toolsets)to register (or a callable for[dynamic toolsets](/docs/ai/tools-toolsets/toolsets#dynamically-building-a-toolset)) | 
| `get_native_tools()` | `Sequence[``AgentNativeTool``]` | [Native tools](/docs/ai/overview/native-tools)to register (including callables) | 
| `get_wrapper_toolset()` | `AbstractToolset`` | None` | [Wrap the agent’s assembled toolset](#toolset-wrapping) | 
| `get_instructions()` | `AgentInstructions`` | None` | [Instructions](/docs/ai/core-concepts/agent#instructions)(static strings,[template strings](/docs/ai/core-concepts/agent-spec#template-strings), or callables) | 
| `get_model_settings()` | `AgentModelSettings`` | None` | [Model settings](/docs/ai/core-concepts/agent#model-run-settings)dict, or a callable for per-step settings | 

Capabilities can hook into five lifecycle points, each with up to four variants:

- `before_*`
- `after_*`
- `wrap_*`- `handler`callable and decides whether/how to call it
- `on_*_error`- `wrap_*`has had its chance to recover), can observe, transform, or recover from errors

| Hook | Signature | Purpose | 
|---|---|---|
| `before_run` | `(ctx: RunContext) -> None` | Observe-only notification that a run is starting | 
| `after_run` | `(ctx: RunContext, *, result: AgentRunResult) -> AgentRunResult` | Modify the final result | 
| `wrap_run` | `(ctx: RunContext, *, handler: WrapRunHandler) -> AgentRunResult` | Wrap the entire run | 
| `on_run_error` | `(ctx: RunContext, *, error: BaseException) -> AgentRunResult` | Handle run errors (see [error hooks](#error-hooks)) | 

`wrap_run` supports error recovery: if `handler()` raises and `wrap_run` catches the exception and returns a result instead, the error is suppressed and the recovery result is used. This works with both [ agent.run()](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run) and 

[.](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter)

`agent.iter()`| Hook | Signature | Purpose | 
|---|---|---|
| `before_node_run` | `(ctx: RunContext, *, node: AgentNode) -> AgentNode` | Observe or replace the node before execution | 
| `after_node_run` | `(ctx: RunContext, *, node: AgentNode, result: NodeResult) -> NodeResult` | Modify the result (next node or `End`) | 
| `wrap_node_run` | `(ctx: RunContext, *, node: AgentNode, handler: WrapNodeRunHandler) -> NodeResult` | Wrap each graph node execution | 
| `on_node_run_error` | `(ctx: RunContext, *, node: AgentNode, error: Exception) -> NodeResult` | Handle node errors (see [error hooks](#error-hooks)) | 

[ wrap_node_run](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_node_run) fires for every node in the 

[agent graph](/docs/ai/core-concepts/agent#iterating-over-an-agents-graph)(

`UserPromptNode`, `ModelRequestNode`, `CallToolsNode`). Override this to observe node transitions, add per-step logging, or modify graph progression:You can also use `wrap_node_run` to modify graph progression — for example, limiting the number of model requests per run:

See [Iterating Over an Agent’s Graph](/docs/ai/core-concepts/agent#iterating-over-an-agents-graph) for more about the agent graph and its node types.

| Hook | Signature | Purpose | 
|---|---|---|
| `before_model_request` | `(ctx: RunContext, request_context: ModelRequestContext) -> ModelRequestContext` | Modify messages, settings, parameters, or model before the model call | 
| `after_model_request` | `(ctx: RunContext, *, request_context: ModelRequestContext, response: ModelResponse) -> ModelResponse` | Modify the model’s response | 
| `wrap_model_request` | `(ctx: RunContext, *, request_context: ModelRequestContext, handler: WrapModelRequestHandler) -> ModelResponse` | Wrap the model call | 
| `on_model_request_error` | `(ctx: RunContext, *, request_context: ModelRequestContext, error: Exception) -> ModelResponse` | Handle model request errors (see [error hooks](#error-hooks)) | 

`ModelRequestContext` bundles `model`, `messages`, `model_settings`, and `model_request_parameters` into a single object, making the signature future-proof. To swap the model for a given request, set `request_context.model` to a different [ Model](/docs/ai/api/models/base/#pydantic_ai.models.Model) instance.

To skip the model call entirely and provide a replacement response, raise [ SkipModelRequest(response)](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.SkipModelRequest) from 

`before_model_request` or `wrap_model_request`.Tool processing has two phases: **validation** (parsing and validating the model’s JSON arguments against the tool’s schema) and **execution** (running the tool function). Each phase has its own hooks.

All tool hooks receive a `tool_def` parameter with the [ ToolDefinition](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition).

**Validation hooks** — `args` is the raw `str | dict[str, Any]` from the model before validation, or the validated `dict[str, Any]` after:

| Hook | Signature | Purpose | 
|---|---|---|
| `before_tool_validate` | `(ctx: RunContext, *, call: ToolCallPart, tool_def: ToolDefinition, args: RawToolArgs) -> RawToolArgs` | Modify raw args before validation (e.g. JSON repair) | 
| `after_tool_validate` | `(ctx: RunContext, *, call: ToolCallPart, tool_def: ToolDefinition, args: ValidatedToolArgs) -> ValidatedToolArgs` | Modify validated args | 
| `wrap_tool_validate` | `(ctx: RunContext, *, call: ToolCallPart, tool_def: ToolDefinition, args: RawToolArgs, handler: WrapToolValidateHandler) -> ValidatedToolArgs` | Wrap the validation step | 
| `on_tool_validate_error` | `(ctx: RunContext, *, call: ToolCallPart, tool_def: ToolDefinition, args: RawToolArgs, error: Exception) -> ValidatedToolArgs` | Handle validation errors (see [error hooks](#error-hooks)) | 

To skip validation and provide pre-validated args, raise [ SkipToolValidation(args)](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.SkipToolValidation) from 

`before_tool_validate` or `wrap_tool_validate`.**Execution hooks** — `args` is always the validated `dict[str, Any]`:

| Hook | Signature | Purpose | 
|---|---|---|
| `before_tool_execute` | `(ctx: RunContext, *, call: ToolCallPart, tool_def: ToolDefinition, args: ValidatedToolArgs) -> ValidatedToolArgs` | Modify args before execution | 
| `after_tool_execute` | `(ctx: RunContext, *, call: ToolCallPart, tool_def: ToolDefinition, args: ValidatedToolArgs, result: Any) -> Any` | Modify execution result | 
| `wrap_tool_execute` | `(ctx: RunContext, *, call: ToolCallPart, tool_def: ToolDefinition, args: ValidatedToolArgs, handler: WrapToolExecuteHandler) -> Any` | Wrap execution | 
| `on_tool_execute_error` | `(ctx: RunContext, *, call: ToolCallPart, tool_def: ToolDefinition, args: ValidatedToolArgs, error: Exception) -> Any` | Handle execution errors (see [error hooks](#error-hooks)) | 

To skip execution and provide a replacement result, raise [ SkipToolExecution(result)](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.SkipToolExecution) from 

`before_tool_execute` or `wrap_tool_execute`.Like tool processing, [output](/docs/ai/core-concepts/output) processing has two phases: **validation** (parsing the model’s raw output against the output schema) and **processing** (extracting the value and calling any [output function](/docs/ai/core-concepts/output#output-functions)). Each phase has its own hooks.

All output hooks receive an `output_context` parameter with [ OutputContext](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.OutputContext) (mode, output type, schema info, and tool call details for 

[tool output](/docs/ai/core-concepts/output#tool-output)).

**Validate hooks** fire only for structured output that requires parsing (prompted, native, tool, union output). They do not fire for plain text or image output. **Process hooks** fire for **all output types** including text, structured, and image output. For [tool output](/docs/ai/core-concepts/output#tool-output), only output hooks fire — tool hooks are skipped entirely.

**Validation hooks** — fire for structured output only; `output` is `str` (raw text) or `dict` (tool args):

| Hook | Signature | Purpose | 
|---|---|---|
| `before_output_validate` | `(ctx, *, output_context, output: RawOutput) -> RawOutput` | Modify raw output before validation (e.g. JSON repair) | 
| `after_output_validate` | `(ctx, *, output_context, output: Any) -> Any` | Modify validated output | 
| `wrap_output_validate` | `(ctx, *, output_context, output: RawOutput, handler) -> Any` | Wrap the validation step | 
| `on_output_validate_error` | `(ctx, *, output_context, output: RawOutput, error: ValidationError | ModelRetry) -> Any` | Handle validation errors (see [error hooks](#error-hooks)) | 

**Processing hooks** — fire for all output types; `output` is the validated/raw output. Output validators ([ @agent.output_validator](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.output_validator)) run inside the processing pipeline (within 

`wrap_output_process`), so `after_output_process` sees the fully validated result:| Hook | Signature | Purpose | 
|---|---|---|
| `before_output_process` | `(ctx, *, output_context, output: Any) -> Any` | Modify output before processing | 
| `after_output_process` | `(ctx, *, output_context, output: Any) -> Any` | Modify processed result | 
| `wrap_output_process` | `(ctx, *, output_context, output: Any, handler) -> Any` | Wrap processing | 
| `on_output_process_error` | `(ctx, *, output_context, output: Any, error: Exception) -> Any` | Handle processing errors (see [error hooks](#error-hooks)) | 

Output validate and process hooks can raise [ ModelRetry](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) to ask the model to try again with a custom message — the same pattern used in 

[output functions](/docs/ai/core-concepts/output#output-functions)and

[output validators](/docs/ai/core-concepts/output#output-validator-functions). See

[Triggering retries with](/docs/ai/core-concepts/hooks#triggering-retries-with-modelretry)for the full pattern.

`ModelRetry`Capabilities can filter or modify which tool definitions the model sees on each step via two hooks:

- `prepare_tools`- **function**tools only. Use this for filtering or modifications to tools the model can call directly.
- `prepare_output_tools`- [output tools](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput)only, with- `ctx.retry`/- `ctx.max_retries`reflecting the- **output**side of the agent retry budget, matching the- [output hook](#output-hooks)lifecycle.

Both hooks operate at the toolset level — the result flows into both the model’s request parameters and `ToolManager.tools`, so filtering also blocks tool execution.

For simple cases, the built-in [ PrepareTools](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrepareTools) / 

[capabilities wrap a callable without a custom subclass.](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrepareOutputTools)

`PrepareOutputTools`For runs with event streaming ([ run_stream_events](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream_events), 

[,](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__)

`event_stream_handler`[UI event streams](/docs/ai/integrations/ui/overview)), capabilities can observe or transform the event stream:

| Hook | Signature | Purpose | 
|---|---|---|
| `wrap_run_event_stream` | `(ctx: RunContext, *, stream: AsyncIterable[AgentStreamEvent]) -> AsyncIterable[AgentStreamEvent]` | Observe, filter, or transform streamed events | 

Matching against [ ToolCallEvent](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallEvent) and 

[handles both function tool calls (](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolResultEvent)

`ToolResultEvent`[/](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.FunctionToolCallEvent)

`FunctionToolCallEvent`[) and output tool calls (](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.FunctionToolResultEvent)

`FunctionToolResultEvent`[/](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.OutputToolCallEvent)

`OutputToolCallEvent`[). Match against the specific subclass when you need to treat them differently.](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.OutputToolResultEvent)

`OutputToolResultEvent`For building web UIs that transform streamed events into protocol-specific formats (like SSE), see the [UI event streams](/docs/ai/integrations/ui/overview) documentation and the [ UIEventStream](/docs/ai/api/ui/base/#pydantic_ai.ui.UIEventStream) base class.

Each lifecycle point has an `on_*_error` hook — the error counterpart to `after_*`. While `after_*` hooks fire on success, `on_*_error` hooks fire on failure (after `wrap_*` has had its chance to recover):

```
before_X → wrap_X(handler)
  ├─ success ─────────→ after_X (modify result)
  └─ failure → on_X_error
        ├─ re-raise ──→ (error propagates, after_X not called)
        └─ recover ───→ after_X (modify recovered result)
```
Error hooks use **raise-to-propagate, return-to-recover** semantics:

- **Raise the original error**— propagates the error unchanged- *(default)*
- **Raise a different exception**— transforms the error
- **Return a result**— suppresses the error and uses the returned value

| Hook | Fires when | Recovery type | 
|---|---|---|
| `on_run_error` | Agent run fails | Return `AgentRunResult` | 
| `on_node_run_error` | Graph node fails | Return next node or `End` | 
| `on_model_request_error` | Model request fails | Return `ModelResponse` | 
| `on_tool_validate_error` | Tool validation fails | Return validated args `dict` | 
| `on_tool_execute_error` | Tool execution fails | Return any tool result | 
| `on_output_validate_error` | Output validation fails | Return validated output | 
| `on_output_process_error` | Output execution fails | Return any output result | 

Capabilities can resolve [deferred tool calls](/docs/ai/tools-toolsets/deferred-tools) — calls that require approval, or that are executed externally — directly from the agent run, without ending the run and waiting for a follow-up:

| Hook | Signature | Purpose | 
|---|---|---|
| `handle_deferred_tool_calls` | `(ctx: RunContext, *, requests: DeferredToolRequests) -> DeferredToolResults | None` | Resolve some or all pending approval/external calls inline | 

Multiple capabilities can each handle a subset: dispatch accumulates results across the chain, passing only the still-unresolved requests to the next capability. Returning `None` (or a [ DeferredToolResults](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) with no entries) declines handling. Anything still unresolved bubbles up as a 

[output for the caller to handle.](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests)

`DeferredToolRequests`For application code that just needs to plug in a handler, use the dedicated [ HandleDeferredToolCalls](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.HandleDeferredToolCalls) capability — see 

[Resolving deferred calls with a handler](/docs/ai/tools-toolsets/deferred-tools#resolving-deferred-calls-with-a-handler).

[ WrapperCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapperCapability) wraps another capability and delegates all methods to it — similar to 

[for toolsets. Subclass it to override specific methods while delegating the rest:](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.WrapperToolset)

`WrapperToolset`The built-in [ PrefixTools](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrefixTools) is an example of a 

`WrapperCapability` — it wraps another capability and prefixes its tool names.By default, a capability instance is shared across all runs of an agent. If your capability accumulates mutable state that should not leak between runs, override [ for_run](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_run) to return a fresh instance:

Capabilities can be built dynamically ahead of each agent run using a function that takes the agent [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and returns a capability or 

`None`. This is useful when the capability — its instructions, model settings, hooks, or contributed toolset — depends on information specific to a run, like its [dependencies](/docs/ai/core-concepts/dependencies).

To register a dynamic capability, pass a function that takes [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) to the 

`capabilities` argument of the [constructor or](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent)

`Agent``agent.run()`. Sync and async functions are both supported. The function is called once per run and the returned capability replaces it for the rest of the run, so its instructions, model settings, toolsets, native tools, and hooks all flow through normally.*(This example is complete, it can be run “as is”)*

To return more than one capability from a single factory, wrap them in a [ CombinedCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CombinedCapability).

When multiple capabilities are passed to an agent, they are composed into a single [ CombinedCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CombinedCapability) that follows 

**middleware semantics**— the same pattern used by web frameworks like Django and Starlette:

- **Configuration**is merged: instructions concatenate, model settings merge additively (later capabilities override earlier ones), toolsets combine, native tools collect.
- `before_*`- `cap1 → cap2 → cap3`.
- `after_*`- `cap3 → cap2 → cap1`.
- `wrap_*`- `cap1`wraps- `cap2`wraps- `cap3`wraps the actual operation. The first capability is the- **outermost**layer.
- `get_wrapper_toolset`

This means the first capability in the list has the first and last say on the operation — it sees the original input before any other capability, and it sees the final output after all inner capabilities have processed it.

By default, capabilities are composed in the order you list them. When a capability needs to be at a specific position regardless of where the user lists it, override [ get_ordering](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_ordering) to return a 

[:](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CapabilityOrdering)

`CapabilityOrdering`The available constraints are:

- `position`- `'outermost'`or- `'innermost'`. Places the capability in a tier before (or after) all capabilities without that position. Multiple capabilities can share a tier; original list order breaks ties within it.
- `wraps`- **type**(matches all instances via- `issubclass`) or a specific- **instance**(matches by identity). Use when your capability needs to see the output of another:- `CapabilityOrdering(wraps=[OtherCapability])`.
- `wrapped_by`- `wraps`. The inverse of- `wraps`.
- `requires`- `UserError`

When constraints are declared, [ CombinedCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CombinedCapability) topologically sorts its children at construction time, preserving user-provided order as a tiebreaker.

[ Hooks](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) supports ordering via the 

`ordering` parameter, so you can declare ordering constraints without subclassing:A guardrail is a capability that intercepts model requests or responses to enforce safety rules. Here’s one that scans model responses for potential PII and redacts it:

The `wrap_*` pattern is useful when you need to observe or time both the input and output of an operation. Here’s a capability that logs every model request and tool call:

[ Pydantic AI Harness](/docs/ai/harness/overview) is the official capability library for Pydantic AI — standalone capabilities like memory, guardrails, context management, and 

[code mode](https://github.com/pydantic/pydantic-ai-harness/tree/main/pydantic_ai_harness/code_mode)live there rather than in core. See

[What goes where?](/docs/ai/harness/overview#what-goes-where)for the full breakdown, or jump to the

[capability matrix](https://github.com/pydantic/pydantic-ai-harness#capability-matrix).

Capabilities are the recommended way for third-party packages to extend Pydantic AI, since they can bundle tools with hooks, instructions, and model settings. See [Extensibility](/docs/ai/guides/extensibility) for the full ecosystem, including [third-party toolsets](/docs/ai/tools-toolsets/toolsets#third-party-toolsets) that can also be wrapped as capabilities.

Capabilities for task planning and progress tracking help agents organize complex work:

- `pydantic-ai-todo`- `TodoCapability`with- `add_todo`,- `read_todos`,- `write_todos`,- `update_todo_status`, and- `remove_todo`tools. Supports subtasks, dependencies, and PostgreSQL persistence. Also available as a lower-level- `TodoToolset`.

Capabilities for managing long conversations help agents stay within context limits:

- `summarization-pydantic-ai`- `ContextManagerCapability`(real-time token tracking, auto-compression at a configurable threshold, and large tool-output truncation);- `SummarizationCapability`(LLM-powered history compression);- `SlidingWindowCapability`(zero-cost message trimming);- `LimitWarnerCapability`(injects a finish-soon hint before hard context limits). Also available as standalone- `history_processors`:- `SummarizationProcessor`,- `SlidingWindowProcessor`, and- `LimitWarnerProcessor`.

Capabilities for spawning and delegating to specialized subagents help agents tackle complex, parallelizable work:

- `subagents-pydantic-ai`- `SubAgentCapability`adds tools for multi-agent delegation:- `task`(spawn a subagent),- `check_task`,- `wait_tasks`,- `list_active_tasks`,- `soft_cancel_task`,- `hard_cancel_task`, and- `answer_subagent`. Supports sync, async, and auto execution modes, nested subagents, and runtime agent creation. Also available as a lower-level toolset via- `create_subagent_toolset`.

Capabilities for cost control, input/output filtering, and tool permissions help keep agents safe and within budget:

- `pydantic-ai-shields`- `CostTracking`(tracks token usage and USD cost per run, raises- `BudgetExceededError`on budget overrun);- `ToolGuard`(block or require approval for specific tools);- `InputGuard`and- `OutputGuard`(custom sync or async validation functions);- `PromptInjection`,- `PiiDetector`,- `SecretRedaction`,- `BlockedKeywords`, and- `NoRefusals`content shields.

Capabilities for filesystem access and sandboxed code execution help agents work with files and run code safely:

- `pydantic-ai-backend`- `ConsoleCapability`registers- `ls`,- `read_file`,- `write_file`,- `edit_file`,- `glob`,- `grep`, and- `execute`tools with a fine-grained permission system. Backends include- `StateBackend`(in-memory, for testing),- `LocalBackend`(real filesystem),- `DockerSandbox`(isolated container execution), and- `CompositeBackend`(routing across backends). Also available as a lower-level- `ConsoleToolset`.

Capabilities that implement [Agent Skills](https://agentskills.io) support help agents efficiently discover and perform specific tasks:

- `pydantic-ai-skills`- `SkillsCapability`implements Agent Skills support with progressive disclosure (load skills on-demand to reduce tokens). Supports filesystem and programmatic skills; compatible with- [agentskills.io](https://agentskills.io).

To add your package to this page, open a pull request.

To make a custom capability usable in [agent specs](/docs/ai/core-concepts/agent-spec), it needs a [ get_serialization_name](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_serialization_name) (defaults to the class name) and a constructor that accepts serializable arguments. The default 

[implementation calls](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.from_spec)

`from_spec``cls(*args, **kwargs)`, so for simple dataclasses no override is needed:Users register custom capability types via the `custom_capability_types` parameter on [ Agent.from_spec](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.from_spec) or 

[.](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.from_file)

`Agent.from_file`Override [ from_spec](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.from_spec) when the constructor takes types that can’t be represented in YAML/JSON. The spec fields should mirror the dataclass fields, but with serializable types:

In YAML this would be `- ConditionalTools: {hidden_tools: [dangerous_tool]}`. In Python code, the full constructor is available: `ConditionalTools(condition=my_check, hidden_tools=['dangerous_tool'])`.

See [Extensibility](/docs/ai/guides/extensibility) for packaging conventions and the broader extension ecosystem.

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/capabilities
