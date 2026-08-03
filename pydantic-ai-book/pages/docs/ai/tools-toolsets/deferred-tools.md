---
type: Web Page
title: Deferred Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Deferred Tools

There are a few scenarios where the model should be able to call a tool that should not or cannot be executed during the same agent run inside the same Python process:

- it may need to be approved by the user first
- it may depend on an upstream service, frontend, or user to provide the result
- the result could take longer to generate than it’s reasonable to keep the agent process running

To support these use cases, Pydantic AI provides the concept of deferred tools, which come in two flavors documented below:

- tools that [require approval](#human-in-the-loop-tool-approval)
- tools that are [executed externally](#external-tool-execution)

When the model calls a deferred tool, there are two ways to resolve it:

- **Resolve it inline** , using a[`HandleDeferredToolCalls`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.HandleDeferredToolCalls)[capability](/docs/ai/capabilities/overview) with a handler that resolves some or all of the pending calls. The agent run continues in a single call without needing to end and restart — use this when the resolver (e.g. an approval gate, an external service client) lives in the same process as the agent. See[Resolving deferred calls with a handler](#resolving-deferred-calls-with-a-handler) .
- **End the run** with a[`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) output object containing information about the deferred tool calls; the caller gathers approvals/results and then starts a new agent run with the original run’s[message history](/docs/ai/core-concepts/message-history) plus a[`DeferredToolResults`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) object. That follow-up is a separate agent run with its own[`run_id`](/docs/ai/core-concepts/message-history#correlating-runs-with-run_id-and-conversation_id) (do not reuse the paused run’s); keep pause/resume correlation via`conversation_id` . Use this when the resolver lives outside the agent process — e.g. a UI adapter that surfaces pending calls to a user and starts a follow-up run once it has their response.

The two flows compose: a handler can resolve a subset of calls and let the rest bubble up as `DeferredToolRequests` output for an outer caller to handle.

The stop-the-world flow requires `DeferredToolRequests` to be in the `Agent`’s [`output_type`](/docs/ai/core-concepts/output#structured-output) so that the possible types of the agent run output are correctly inferred. If your agent can also be used in a context where no deferred tools are available and you don’t want to deal with that type everywhere you use the agent, you can instead pass the `output_type` argument when you run the agent using [`agent.run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run), [`agent.run_sync()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_sync), [`agent.run_stream()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream), or [`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter). Note that the run-time `output_type` overrides the one specified at construction time (for type inference reasons), so you’ll need to include the original output type explicitly.

The recommended way to handle deferred tool calls is to register a `HandleDeferredToolCalls`[capability](/docs/ai/capabilities/overview) whose handler receives the [`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) and returns a [`DeferredToolResults`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) resolving some or all of them. The tool execution pipeline applies the results inline and the agent run continues in a single call, as if the deferred tools had returned normally.

With a handler in place, `DeferredToolRequests` no longer needs to be declared as an output type — unless you also want unresolved calls to bubble up to the caller (see below).

[`DeferredToolRequests.build_results()`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests.build_results) is a convenience constructor — it validates that every tool call ID refers to a pending request of the correct kind, and accepts `approve_all=True` to auto-approve any approval requests not otherwise specified.

Never reached here — the handler denies this call, so the model sees the denial message instead.

The handler supplies the result for this external call, so the tool body just signals the deferral.

If the handler declines to resolve some or all of the calls (by omitting them from the returned [`DeferredToolResults`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) or returning `None`), the next [`HandleDeferredToolCalls`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.HandleDeferredToolCalls) (or any other capability that overrides the [`handle_deferred_tool_calls`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.handle_deferred_tool_calls) hook) gets a chance, and any still-unresolved calls bubble up as a [`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) output. To allow that bubble-up, include `DeferredToolRequests` in the agent’s `output_type` — so you can combine inline handling with the stop-the-world flow when it makes sense.

If you’re [building a custom capability](/docs/ai/capabilities/custom) that needs to resolve approvals or external calls itself (e.g. a sandbox that exposes deferred tools), override the [`handle_deferred_tool_calls`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.handle_deferred_tool_calls) hook directly on your capability instead of registering a separate `HandleDeferredToolCalls`. The same hook is also available via the [`Hooks`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) capability — see [Hooks](/docs/ai/core-concepts/hooks#deferred-tool-call-hook).

The sections below describe the two kinds of deferred tools the handler can resolve, as well as the alternative stop-the-world flow for each. See [Capabilities](/docs/ai/capabilities/overview) for how multiple capabilities compose, including [`WrapperCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapperCapability) and the `capabilities=[...]` list.

If a tool function always requires approval, you can pass the `requires_approval=True` argument to the [`@agent.tool`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool) decorator, [`@agent.tool_plain`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain) decorator, [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) class, [`FunctionToolset.tool`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.tool) decorator, or [`FunctionToolset.add_function()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.add_function) method. Inside the function, you can then assume that the tool call has been approved.

If whether a tool function requires approval depends on the tool call arguments or the agent [run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) (e.g. [dependencies](/docs/ai/core-concepts/dependencies) or message history), you can raise the [`ApprovalRequired`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ApprovalRequired) exception from the tool function. The [`RunContext.tool_call_approved`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.tool_call_approved) property will be `True` if the tool call has already been approved.

You can also raise it from the tool’s [`args_validator`](/docs/ai/tools-toolsets/tools-advanced#args-validator), which runs before the tool function and lets you reject invalid arguments before asking a human to approve them.

To require approval for calls to tools provided by a [toolset](/docs/ai/tools-toolsets/toolsets) (like an [MCP server](/docs/ai/mcp/client)), see the [`ApprovalRequiredToolset` documentation](/docs/ai/tools-toolsets/toolsets#requiring-tool-approval).

When the model calls a tool that requires approval, the agent run will end with a [`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) output object with an `approvals` list holding [`ToolCallPart`s](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart) containing the tool name, validated arguments, and a unique tool call ID.

Once you’ve gathered the user’s approvals or denials, you can build a [`DeferredToolResults`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) object with an `approvals` dictionary that maps each tool call ID to a boolean, a [`ToolApproved`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolApproved) object (with optional `override_args`), or a [`ToolDenied`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDenied) object (with an optional custom `message` to provide to the model). You can also provide a `metadata` dictionary on `DeferredToolResults` that maps each tool call ID to a dictionary of metadata that will be available in the tool’s [`RunContext.tool_call_metadata`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.tool_call_metadata) attribute. This `DeferredToolResults` object can then be provided to one of the agent run methods as `deferred_tool_results`, alongside the original run’s [message history](/docs/ai/core-concepts/message-history).

Here’s an example that shows how to require approval for all file deletions, and for updates of specific protected files:

The optional `metadata` parameter can attach arbitrary context to deferred tool calls, accessible in `DeferredToolRequests.metadata` keyed by `tool_call_id`.

This second agent run continues from where the first run left off, providing the tool approval results and optionally a new `user_prompt` to give the model additional instructions alongside the deferred results.

*(This example is complete, it can be run “as is”)*

When the result of a tool call cannot be generated inside the same agent run in which it was called, the tool is considered to be external. Examples of external tools are client-side tools implemented by a web or app frontend, and slow tasks that are passed off to a background worker or external service instead of keeping the agent process running.

If whether a tool call should be executed externally depends on the tool call arguments, the agent [run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) (e.g. [dependencies](/docs/ai/core-concepts/dependencies) or message history), or how long the task is expected to take, you can define a tool function and conditionally raise the [`CallDeferred`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.CallDeferred) exception. Before raising the exception, the tool function would typically schedule some background task and pass along the [`RunContext.tool_call_id`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.tool_call_id) so that the result can be matched to the deferred tool call later.

As with approval, the tool’s [`args_validator`](/docs/ai/tools-toolsets/tools-advanced#args-validator) can raise `CallDeferred` too, so only calls with valid arguments are handed off.

If a tool is always executed externally and its definition is provided to your code along with a JSON schema for its arguments, you can use an [`ExternalToolset`](/docs/ai/tools-toolsets/toolsets#external-toolset). If the external tools are known up front and you don’t have the arguments JSON schema handy, you can also define a tool function with the appropriate signature that does nothing but raise the [`CallDeferred`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.CallDeferred) exception.

When the model calls an external tool, the agent run will end with a [`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) output object with a `calls` list holding [`ToolCallPart`s](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart) containing the tool name, validated arguments, and a unique tool call ID.

Once the tool call results are ready, you can build a [`DeferredToolResults`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) object with a `calls` dictionary that maps each tool call ID to an arbitrary value to be returned to the model, a [`ToolReturn`](/docs/ai/tools-toolsets/tools-advanced#advanced-tool-returns) object, or an exception in case the tool call failed: a [`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) if the model should [try again](/docs/ai/tools-toolsets/tools-advanced#tool-retries), or a [`ToolFailed`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ToolFailed) if the failure should be reported to the model as a [failed result](/docs/ai/tools-toolsets/tools-advanced#tool-failed) (without consuming the tool’s retry budget) so it can decide how to proceed. This `DeferredToolResults` object can then be provided to one of the agent run methods as `deferred_tool_results`, alongside the original run’s [message history](/docs/ai/core-concepts/message-history).

Here’s an example that shows how to move a task that takes a while to complete to the background and return the result to the model once the task is complete:

Generate a task ID that can be tracked independently of the tool call ID.

The optional `metadata` parameter passes the `task_id` so it can be matched with results later, accessible in `DeferredToolRequests.metadata` keyed by `tool_call_id`.

In reality, this would typically happen in a separate process that polls for the task status or is notified when all pending tasks are complete.

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

Like any other tool call, a deferred tool call emits a [`FunctionToolCallEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.FunctionToolCallEvent) into the [event stream](/docs/ai/core-concepts/agent#streaming-events-and-final-output) — but that event alone doesn’t tell a stream consumer that the call is paused waiting for interaction, or what kind of interaction is expected. Two additional [`AgentStreamEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AgentStreamEvent)s carry that context:

- [`DeferredToolRequestsEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.DeferredToolRequestsEvent) — emitted once per batch of deferred calls, carrying the[`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) . It’s emitted before any[`HandleDeferredToolCalls`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.HandleDeferredToolCalls) handler runs, so a consumer can for example notify a frontend that input is needed while the handler waits for it. If no handler resolves all of the requests, the run ends with the pending requests as its`DeferredToolRequests` output.
- [`DeferredToolResultsEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.DeferredToolResultsEvent) — emitted when a handler resolves (some of) the requests, carrying the[`DeferredToolResults`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) . The resolved calls then execute through the regular pipeline, emitting a[`FunctionToolResultEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.FunctionToolResultEvent) each. No event is emitted when results are instead provided to a new run via`deferred_tool_results` , as in that case the caller already knows them.

This keeps resolution and presentation decoupled: a handler can contain pure resolution logic (e.g. waiting for a signal in a [durable execution](/docs/ai/capabilities/durable_execution/overview) workflow), while a stream consumer owns all communication with the frontend, without maintaining its own mapping of which tools are interactive.

Continuing the [handler example](#resolving-deferred-calls-with-a-handler) from above:

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

- [Function Tools](/docs/ai/tools-toolsets/tools) - Basic tool concepts and registration
- [Advanced Tool Features](/docs/ai/tools-toolsets/tools-advanced) - Custom schemas, dynamic tools, and execution details
- [Toolsets](/docs/ai/tools-toolsets/toolsets) - Managing collections of tools, including`ExternalToolset` for external tools
- [Message History](/docs/ai/core-concepts/message-history) - Working with message history for deferred tools, including[`run_id` / `conversation_id`](/docs/ai/core-concepts/message-history#correlating-runs-with-run_id-and-conversation_id)

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools
