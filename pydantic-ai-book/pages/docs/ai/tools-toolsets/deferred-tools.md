---
type: Web Page
title: Deferred Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Deferred Tools

There are a few scenarios where the model should be able to call a tool that should not or cannot be executed during the same agent run inside the same Python process:

- it may need to be approved by the user first
- it may depend on an upstream service, frontend, or user to provide the result
- the result could take longer to generate than it’s reasonable to keep the agent process running

To support these use cases, Pydantic AI provides the concept of deferred tools, which come in two flavors documented below:

- tools that require approval
- tools that are executed externally

When the model calls a deferred tool, there are two ways to resolve it:

- **Resolve it inline**, using a- `HandleDeferredToolCalls`capability with a handler that resolves some or all of the pending calls. The agent run continues in a single call without needing to end and restart — use this when the resolver (e.g. an approval gate, an external service client) lives in the same process as the agent. See Resolving deferred calls with a handler.
- **End the run**with a- `DeferredToolRequests`output object containing information about the deferred tool calls; the caller gathers approvals/results and then starts a new agent run with the original run’s message history plus a- `DeferredToolResults`object. Use this when the resolver lives outside the agent process — e.g. a UI adapter that surfaces pending calls to a user and starts a follow-up run once it has their response.

The two flows compose: a handler can resolve a subset of calls and let the rest bubble up as `DeferredToolRequests` output for an outer caller to handle.

The stop-the-world flow requires `DeferredToolRequests` to be in the `Agent`’s `output_type` so that the possible types of the agent run output are correctly inferred. If your agent can also be used in a context where no deferred tools are available and you don’t want to deal with that type everywhere you use the agent, you can instead pass the `output_type` argument when you run the agent using `agent.run()`, `agent.run_sync()`, `agent.run_stream()`, or `agent.iter()`. Note that the run-time `output_type` overrides the one specified at construction time (for type inference reasons), so you’ll need to include the original output type explicitly.

The recommended way to handle deferred tool calls is to register a `HandleDeferredToolCalls` capability whose handler receives the `DeferredToolRequests` and returns a `DeferredToolResults` resolving some or all of them. The tool execution pipeline applies the results inline and the agent run continues in a single call, as if the deferred tools had returned normally.

With a handler in place, `DeferredToolRequests` no longer needs to be declared as an output type — unless you also want unresolved calls to bubble up to the caller (see below).

`DeferredToolRequests.build_results()` is a convenience constructor — it validates that every tool call ID refers to a pending request of the correct kind, and accepts `approve_all=True` to auto-approve any approval requests not otherwise specified.

Never reached here — the handler denies this call, so the model sees the denial message instead.

The handler supplies the result for this external call, so the tool body just signals the deferral.

If the handler declines to resolve some or all of the calls (by omitting them from the returned `DeferredToolResults` or returning `None`), the next `HandleDeferredToolCalls` (or any other capability that overrides the `handle_deferred_tool_calls` hook) gets a chance, and any still-unresolved calls bubble up as a `DeferredToolRequests` output. To allow that bubble-up, include `DeferredToolRequests` in the agent’s `output_type` — so you can combine inline handling with the stop-the-world flow when it makes sense.

If you’re building a custom capability that needs to resolve approvals or external calls itself (e.g. a sandbox that exposes deferred tools), override the `handle_deferred_tool_calls` hook directly on your capability instead of registering a separate `HandleDeferredToolCalls`. The same hook is also available via the `Hooks` capability — see Hooks.

The sections below describe the two kinds of deferred tools the handler can resolve, as well as the alternative stop-the-world flow for each. See Capabilities for how multiple capabilities compose, including `WrapperCapability` and the `capabilities=[...]` list.

If a tool function always requires approval, you can pass the `requires_approval=True` argument to the `@agent.tool` decorator, `@agent.tool_plain` decorator, `Tool` class, `FunctionToolset.tool` decorator, or `FunctionToolset.add_function()` method. Inside the function, you can then assume that the tool call has been approved.

If whether a tool function requires approval depends on the tool call arguments or the agent run context (e.g. dependencies or message history), you can raise the `ApprovalRequired` exception from the tool function. The `RunContext.tool_call_approved` property will be `True` if the tool call has already been approved.

To require approval for calls to tools provided by a toolset (like an MCP server), see the `ApprovalRequiredToolset` documentation.

When the model calls a tool that requires approval, the agent run will end with a `DeferredToolRequests` output object with an `approvals` list holding `ToolCallPart`s containing the tool name, validated arguments, and a unique tool call ID.

Once you’ve gathered the user’s approvals or denials, you can build a `DeferredToolResults` object with an `approvals` dictionary that maps each tool call ID to a boolean, a `ToolApproved` object (with optional `override_args`), or a `ToolDenied` object (with an optional custom `message` to provide to the model). You can also provide a `metadata` dictionary on `DeferredToolResults` that maps each tool call ID to a dictionary of metadata that will be available in the tool’s `RunContext.tool_call_metadata` attribute. This `DeferredToolResults` object can then be provided to one of the agent run methods as `deferred_tool_results`, alongside the original run’s message history.

Here’s an example that shows how to require approval for all file deletions, and for updates of specific protected files:

The optional `metadata` parameter can attach arbitrary context to deferred tool calls, accessible in `DeferredToolRequests.metadata` keyed by `tool_call_id`.

This second agent run continues from where the first run left off, providing the tool approval results and optionally a new `user_prompt` to give the model additional instructions alongside the deferred results.

*(This example is complete, it can be run “as is”)*

When the result of a tool call cannot be generated inside the same agent run in which it was called, the tool is considered to be external. Examples of external tools are client-side tools implemented by a web or app frontend, and slow tasks that are passed off to a background worker or external service instead of keeping the agent process running.

If whether a tool call should be executed externally depends on the tool call arguments, the agent run context (e.g. dependencies or message history), or how long the task is expected to take, you can define a tool function and conditionally raise the `CallDeferred` exception. Before raising the exception, the tool function would typically schedule some background task and pass along the `RunContext.tool_call_id` so that the result can be matched to the deferred tool call later.

If a tool is always executed externally and its definition is provided to your code along with a JSON schema for its arguments, you can use an `ExternalToolset`. If the external tools are known up front and you don’t have the arguments JSON schema handy, you can also define a tool function with the appropriate signature that does nothing but raise the `CallDeferred` exception.

When the model calls an external tool, the agent run will end with a `DeferredToolRequests` output object with a `calls` list holding `ToolCallPart`s containing the tool name, validated arguments, and a unique tool call ID.

Once the tool call results are ready, you can build a `DeferredToolResults` object with a `calls` dictionary that maps each tool call ID to an arbitrary value to be returned to the model, a `ToolReturn` object, or a `ModelRetry` exception in case the tool call failed and the model should try again. This `DeferredToolResults` object can then be provided to one of the agent run methods as `deferred_tool_results`, alongside the original run’s message history.

Here’s an example that shows how to move a task that takes a while to complete to the background and return the result to the model once the task is complete:

Generate a task ID that can be tracked independently of the tool call ID.

The optional `metadata` parameter passes the `task_id` so it can be matched with results later, accessible in `DeferredToolRequests.metadata` keyed by `tool_call_id`.

In reality, this would typically happen in a separate process that polls for the task status or is notified when all pending tasks are complete.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

- Function Tools - Basic tool concepts and registration
- Advanced Tool Features - Custom schemas, dynamic tools, and execution details
- Toolsets - Managing collections of tools, including `ExternalToolset`for external tools
- Message History - Understanding how to work with message history for deferred tools

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools
