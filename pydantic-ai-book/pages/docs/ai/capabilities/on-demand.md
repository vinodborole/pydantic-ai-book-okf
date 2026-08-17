---
type: Web Page
title: On-Demand Capabilities | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/on-demand
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# On-Demand Capabilities

A capability is a bundle of instructions and/or tools, optionally with settings and hooks. A multi-workflow agent normally sends every workflow’s instructions and tool schemas on every turn, and applies every workflow’s settings and hooks for the whole run — even though most requests need just one workflow. That cost grows with each workflow you add: more input tokens, and worse tool selection once the visible tool set passes the ~30–50-tool mark where models start picking the wrong one (the same pressure behind [tool search](/docs/ai/tools-toolsets/tools-advanced/#tool-search)).

Mark a [capability](/docs/ai/capabilities/overview/) with `defer_loading=True` and give it a stable `id`, and it collapses to a one-line catalog entry — its `id` plus an optional `description` — that the model pulls in on demand. Here’s the minimal shape:

On the first turn, the refund workflow is collapsed to a catalog entry. The model sees its base instructions, the framework-managed `load_capability` tool, and the catalog appended to the instructions:

```
The following capabilities are deferred and can be loaded using the `load_capability` tool. A capability may have tools; they stay hidden until it is loaded:
- refunds: Use for refund eligibility, refund status, or processing a refund.
```
The model does not receive the refund instructions or the `refund_status` tool definition yet, so it has no reason to call the tool. Depending on the active model, Pydantic AI may also send provider/tool-search plumbing to preserve the hidden state; that plumbing does not expose the refund tool definition until the capability is loaded. The exchange unfolds across model requests within a single `agent.run_sync` call:

1. **Request 1.** The model sees the catalog above and the user’s prompt. It calls the`load_capability` tool with`id='refunds'` .
2. **Load.** Pydantic AI returns the capability’s instructions —*“Always confirm the order ID before issuing a refund.”* — as the tool result and exposes the`refund_status` definition on the next request.
3. **Request 2.** The model now sees those instructions in history and`refund_status` in its tool list. It calls`refund_status(order_id='ABC-123')` and answers the user from the result.

Already-loaded capabilities stay loaded for the rest of the run — the model never needs to re-open one.

Searching cannot reveal a capability-owned tool: it stays hidden until its capability loads. In runs that also have searchable deferred tools, the catalog explicitly steers the model to load the capability rather than search for its tools; in capability-only runs — where no search surface exists — the catalog omits any mention of searching.

Loading activates the whole bundle, not just instructions: the capability’s function tools, model settings, and lifecycle hooks come live together (see [What you can defer](#what-you-can-defer)). It’s a one-line change to a capability you already register, it works on [every provider](#cross-provider-behavior), and it [survives history replay](#resumable-across-runs).

Every part of a capability bundle activates together as a single unit:

| Part | Before load | After load | 
|---|---|---|
| Instructions (static or dynamic) | Not sent | Returned as the `load_capability` tool result; included in subsequent requests | 
| Function tools | Not exposed | Exposed on the next request | 
| Model settings (static or per-step) | Not applied | Merged into the run’s settings for subsequent requests | 
| Lifecycle [hooks](/docs/ai/capabilities/custom/#hooking-into-the-lifecycle) | Do not fire | Fire after the capability is loaded | 
| [Native tools](/docs/ai/tools-toolsets/native-tools/) | Not exposed | Exposed on the next request — see [Cache implications](#cache-implications) | 

**Reach for on-demand capabilities when:**

- the agent serves multiple distinct workflows (refunds, returns, fraud review, account security…) where most turns need one
- a workflow needs *more than instructions* — its own tools, raised reasoning effort, an approval hook — and those should travel together as a unit
- you want skills-style progressive disclosure but also want the loaded bundle to bring tools and settings, not just a runbook

**Skip it when:**

- the capability is used on most turns — the discovery round-trip costs more than the tokens it saves
- you have a flat catalog of individually-discoverable tools with no shared instructions — use [tool search](/docs/ai/tools-toolsets/tools-advanced/#tool-search) instead, which discovers individual tools by name rather than loading bundles

If you’ve used [Anthropic’s Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills), this is the same idea generalised: a skill is a markdown file the model can pull in on demand. An on-demand capability does that *plus* typed function tools, per-step model settings, and lifecycle hooks.

`defer_loading=True` is not specific to the [`Capability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) convenience class. The shared fields live on [`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability), and built-in capabilities expose `id`, `description`, and `defer_loading` on construction. For custom capabilities, set those attributes on the instance.

Until the model loads `analytics-mcp`, none of the MCP server’s tool definitions enter the prompt. The same flag works on [`WebSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebSearch), [`WebFetch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WebFetch), [`Hooks`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks), and any custom [`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) subclass — see [Building custom capabilities](/docs/ai/capabilities/custom/) for adding `defer_loading` to your own subclass.

Loaded-capability and tool-availability state live in message history, not in the agent. When a conversation is persisted to a database and resumed later — possibly on a different process, machine, or model — Pydantic AI reconstructs the loaded capability IDs from `load_capability` call/return pairs and the revealed tool names from [`ToolAvailabilityDeltaPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolAvailabilityDeltaPart). Capabilities the model loaded earlier stay loaded; capabilities it never loaded stay collapsed in the catalog. No re-discovery round-trip on resume.

This is why deferred capabilities require a stable explicit `id`: history replay matches calls to capabilities by id, so a class-derived id would silently break the moment a class is renamed. The same property makes cross-provider replay work — a run that loaded `refunds` on Anthropic and continued on OpenAI Responses keeps `refunds` loaded after the switch.

History carries *which* capability ids were loaded, not the capabilities themselves: the resuming agent must be constructed with the same capabilities (matching `id`s), just as it must be constructed with the same tools. State lives in history; definitions live in code.

Several [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) fields expose progressive-disclosure state to tools, hooks, and capability-owned callbacks:

- `ctx.loaded_capability_ids` — deferred capability IDs explicitly loaded through the`load_capability` tool, reconstructed from message history before each model request. A capability loaded during a step appears from the*next* step onwards, which is also the first step on which its instructions and tools reach the model.
- `ctx.available_capability_ids` — the currently-live capability IDs: always-available capabilities plus`ctx.loaded_capability_ids` .
- `ctx.capability_loaded` — only meaningful while Pydantic AI is running a capability-owned hook or callback. It is scoped to that capability; deferred hooks and callbacks are skipped until this value would be true.
- `ctx.discovered_tool_names` — deferred function tools revealed by durable history, whether through tool search,[`ToolReturn.tools`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolReturn) , or a capability load.
- `ctx.available_tool_names` — function tool names currently known as available: always-visible tools from the current step’s assembled tool manager plus names revealed in history. Early hooks such as`before_run` may see only the history-derived names, or an empty set if none exist yet, before tool definitions have been prepared. See[Hook ordering](/docs/ai/core-concepts/hooks/#hook-ordering) for how hook timing affects what is populated.
- `ctx.is_tool_available(tool)` — whether a function tool is currently visible. Wrapping toolsets should pass the[`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition) they hold; model-request hooks and tool execution can pass a name from the current`ctx.tools` snapshot.
- `ctx.usage_limits` — the[`UsageLimits`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.UsageLimits) the run is enforcing (defaulting to`UsageLimits()` when none were passed, so it’s only`None` outside of a run), alongside`ctx.usage` for the usage so far. A capability can read the run’s limits to disclose or adapt to the remaining budget (e.g. budget disclosure) without being configured with a duplicate copy. Treat it as read-only: it’s the live object the run enforces against, so mutating a field would change what the run enforces on subsequent requests.

Loading a capability updates the capability state immediately, but the loaded bundle’s function tools, native tools, and model settings take effect on the next model request.

On-demand capabilities work on every model, and where the provider can express an availability change natively, loading one leaves the prompt prefix intact.

A capability-owned tool is hidden until its capability loads, and it is never searchable — the model reaches it by loading the capability, not by asking for it. The unified rule is that an unrevealed deferred tool stays outside the model’s usable context; each provider’s reveal mechanism determines its wire representation.

- **Anthropic `tool_addition_mode='by_reference'`** references the revealed name in a`tool_addition` block. A capability-only run pre-advertises the definition with`defer_loading=True` ; a mixed run with a search surface withholds it, then appends the deferred definition in the same request as the reveal.
- **OpenAI Responses `tool_addition_mode='with_definitions'`** carries the full revealed definition in an appended`additional_tools` input item and leaves it out of`tools` .
- **No provider-native reveal-item support (`tool_addition_mode=None`)** announces`The following tool(s) are now available: {names}` when the schema is visible. It synthesizes a`search_tools` exchange only when a result must reveal a schema that is still withheld.

Add a standalone `defer_loading=True` tool to the same run and tool search comes back for it, since that one genuinely is searchable. Capability-owned tools stay off the wire entirely while a search surface is present, so search remains fully native — server-executed where the model supports it — and no query can surface a tool whose capability has not loaded.

Calling the `load_capability` tool reveals capability behavior between requests. Whether that breaks the provider’s prompt-cache prefix depends on what’s revealed:

`load_capability` returns the loaded capability’s function-tool names through [`ToolReturn.tools`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolReturn), and the executor records the availability delta beside the tool result. Any user tool can use the same source. Histories that contain a complete capability-load exchange without its delta are translated before the next model request.

| What loads | Cache prefix | 
|---|---|
| Instructions only | **Stable** — instructions land in the message history, not the request prefix. | 
| Function tools with provider-native reveal-item support ( `tool_addition_mode='by_reference'` or`'with_definitions'` ) | **Stable on Anthropic and OpenAI Responses** — deferred Anthropic entries are outside its cache key, and OpenAI Responses appends`additional_tools` without changing`tools[]` . | 
| Function tools without provider-native reveal-item support ( `tool_addition_mode=None` ) | **May break between turns** — function-tool visibility can change as capabilities load. | 
| Native tools | **Always breaks the prefix on load** — native tool definitions are part of the request prefix on every provider. | 

When preserving the cache prefix matters, prefer instruction-only or function-tool-only on-demand capabilities on a model that can express an availability change natively. The provider-specific mechanics that keep the prefix stable live in [Tool search and prompt caching](/docs/ai/tools-toolsets/tools-advanced/#tool-search-caching).

[`Capability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) bundles instructions, function tools, and toolsets without subclassing. Register tools with the decorator that mirrors [`@agent.tool`](/docs/ai/tools-toolsets/tools/#registering-function-tools-via-decorator):

In addition to `@capability.tool` and `@capability.tool_plain`, you can pass existing functions or [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) instances via `tools=`, or hand in one or more [toolsets](/docs/ai/tools-toolsets/toolsets/) via `toolsets=`. For dynamic instructions, use the [`@capability.instructions`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability.instructions) decorator. For a dynamic catalog entry, pass a callable as `description=`.

`@capability.tool` and `@capability.tool_plain` mirror [`@agent.tool`](/docs/ai/tools-toolsets/tools/#registering-function-tools-via-decorator) exactly, including the `defer_loading` argument. On a deferred capability that per-tool flag is a no-op — the capability gates all its tools as a unit — so it only has an effect on a non-deferred `Capability`, where it opts an individual tool into [tool search](/docs/ai/tools-toolsets/tools-advanced/#tool-search) discovery.

For anything beyond instructions, function tools, toolsets, and descriptions — model settings, hooks, native tools, wrapper toolsets, or custom per-run logic — subclass [`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) directly. When subclassing, override [`get_description`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_description) if the catalog entry needs to vary by run.

The [`Capability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) example above deferred instructions and a function tool, but the same flag gates the whole bundle — what the model knows, what it can do, and how it does it (see [What you can defer](#what-you-can-defer)). The snippets below show the remaining pieces in turn: model settings, hooks, and native tools.

[`get_model_settings`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model_settings) is collected during capability assembly, but its settings are only applied after the deferred capability is loaded. That means per-step settings like raised reasoning effort only apply for workflows the model opts into:

Hooks can live on deferred capabilities too. They do not run until the model loads the capability that owns them:

Any [provider-adaptive capability](/docs/ai/capabilities/overview/#provider-adaptive-tools) (`WebSearch`, `WebFetch`, `MCP`, …) can be deferred the same way. The native tool definition only enters the request after the `load_capability` tool loads the capability — see [Cache implications](#cache-implications) for the trade-off:

A realistic on-demand capability rarely consists of just one piece. The example below defines a customer-support agent with two deferred workflows that exercise different parts of the bundle:

- `orders` — instructions plus a function tool, defined inline with[`Capability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) .
- `account-security` — instructions, a function tool, raised reasoning effort,*and* an approval hook, all bundled as one[`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) subclass.

For those workflows, turn 1 exposes only the two-line catalog. Base instructions, always-on tools, the framework-managed `load_capability` tool, and any provider/tool-search plumbing still appear as usual. Loading `account-security` activates the runbook, the destructive tool, the higher reasoning effort, *and* the approval gate together — that’s what we mean by bundle-level disclosure.

A “where is my order?” request loads only `orders`. A “someone is logging into my account” request loads only `account-security` — and from that point on, every tool call in the run passes through the approval hook *and* benefits from the raised reasoning effort, without either being visible to the model on requests that never touched the workflow.

Want the model to actually *read the runbook* before taking a destructive action? Make the runbook a deferred capability, then check `ctx.loaded_capability_ids` in a one-method hook:

The model sees `issue_refund` from turn 1. If it tries to call it before opening `refund-policy`, the hook bounces the call back with a message pointing at the exact `load_capability` tool call to make. The model loads the policy, the policy text lands in its recent context, and the refund runs *within* the rules — and only then. Same shape for any tool-and-runbook pair.

Because the loaded set is just runtime data on [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext), the pattern generalises: dynamic instructions can warn when a risky pair of workflows is open, audit hooks can tag traces with the loaded set, escalation hooks can require an extra confirmation when both `payments` and `account-security` are active.

If you already keep your skills as Markdown files with YAML frontmatter — the format used by [Anthropic Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) — you can wrap each one in a [`Capability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) with a few lines of glue.

Given a skill file `skills/refunds.md`:

Load it into an agent as an on-demand capability:

Each file shows up in the model’s catalog as its `id` plus `description`; the body is only sent once the model calls the `load_capability` tool. To go beyond instructions — add function tools, model settings, or hooks for a particular skill — subclass [`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) as in the examples above.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/on-demand
