---
type: Web Page
title: Deferred Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools
timestamp: '2026-07-09T12:16:42.049694+00:00'
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

- **Resolve it inline**, using a- `HandleDeferredToolCalls`- [capability](/docs/ai/core-concepts/capabilities)with a handler that resolves some or all of the pending calls. The agent run continues in a single call without needing to end and restart — use this when the resolver (e.g. an approval gate, an external service client) lives in the same process as the agent. See- [Resolving deferred calls with a handler](#resolving-deferred-calls-with-a-handler).
- **End the run**with a- `DeferredToolRequests`- [message history](/docs/ai/core-concepts/message-history)plus a- `DeferredToolResults`

The two flows compose: a handler can resolve a subset of calls and let the rest bubble up as `DeferredToolRequests` output for an outer caller to handle.

The stop-the-world flow requires `DeferredToolRequests` to be in the `Agent`’s [ output_type](/docs/ai/core-concepts/output#structured-output) so that the possible types of the agent run output are correctly inferred. If your agent can also be used in a context where no deferred tools are available and you don’t want to deal with that type everywhere you use the agent, you can instead pass the 

`output_type` argument when you run the agent using [,](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run)

`agent.run()`[,](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_sync)

`agent.run_sync()`[, or](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream)

`agent.run_stream()`[. Note that the run-time](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter)

`agent.iter()``output_type` overrides the one specified at construction time (for type inference reasons), so you’ll need to include the original output type explicitly.The recommended way to handle deferred tool calls is to register a `HandleDeferredToolCalls`[capability](/docs/ai/core-concepts/capabilities) whose handler receives the [ DeferredToolRequests](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) and returns a 

[resolving some or all of them. The tool execution pipeline applies the results inline and the agent run continues in a single call, as if the deferred tools had returned normally.](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults)

`DeferredToolResults`With a handler in place, `DeferredToolRequests` no longer needs to be declared as an output type — unless you also want unresolved calls to bubble up to the caller (see below).

[ DeferredToolRequests.build_results()](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests.build_results) is a convenience constructor — it validates that every tool call ID refers to a pending request of the correct kind, and accepts 

`approve_all=True` to auto-approve any approval requests not otherwise specified.Never reached here — the handler denies this call, so the model sees the denial message instead.

The handler supplies the result for this external call, so the tool body just signals the deferral.

If the handler declines to resolve some or all of the calls (by omitting them from the returned [ DeferredToolResults](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) or returning 

`None`), the next [(or any other capability that overrides the](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.HandleDeferredToolCalls)

`HandleDeferredToolCalls`[hook) gets a chance, and any still-unresolved calls bubble up as a](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.handle_deferred_tool_calls)

`handle_deferred_tool_calls`[output. To allow that bubble-up, include](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests)

`DeferredToolRequests``DeferredToolRequests` in the agent’s `output_type` — so you can combine inline handling with the stop-the-world flow when it makes sense.If you’re [building a custom capability](/docs/ai/core-concepts/capabilities#building-custom-capabilities) that needs to resolve approvals or external calls itself (e.g. a sandbox that exposes deferred tools), override the [ handle_deferred_tool_calls](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.handle_deferred_tool_calls) hook directly on your capability instead of registering a separate 

`HandleDeferredToolCalls`. The same hook is also available via the [capability — see](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks)

`Hooks`[Hooks](/docs/ai/core-concepts/hooks#deferred-tool-call-hook).

The sections below describe the two kinds of deferred tools the handler can resolve, as well as the alternative stop-the-world flow for each. See [Capabilities](/docs/ai/core-concepts/capabilities) for how multiple capabilities compose, including [ WrapperCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapperCapability) and the 

`capabilities=[...]` list.If a tool function always requires approval, you can pass the `requires_approval=True` argument to the [ @agent.tool](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool) decorator, 

[decorator,](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain)

`@agent.tool_plain`[class,](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool)

`Tool`[decorator, or](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.tool)

`FunctionToolset.tool`[method. Inside the function, you can then assume that the tool call has been approved.](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.add_function)

`FunctionToolset.add_function()`If whether a tool function requires approval depends on the tool call arguments or the agent [run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) (e.g. [dependencies](/docs/ai/core-concepts/dependencies) or message history), you can raise the [ ApprovalRequired](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ApprovalRequired) exception from the tool function. The 

[property will be](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.tool_call_approved)

`RunContext.tool_call_approved``True` if the tool call has already been approved.To require approval for calls to tools provided by a [toolset](/docs/ai/tools-toolsets/toolsets) (like an [MCP server](/docs/ai/mcp/client)), see the [ ApprovalRequiredToolset documentation](/docs/ai/tools-toolsets/toolsets#requiring-tool-approval).

When the model calls a tool that requires approval, the agent run will end with a [ DeferredToolRequests](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) output object with an 

`approvals` list holding [containing the tool name, validated arguments, and a unique tool call ID.](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)

`ToolCallPart`sOnce you’ve gathered the user’s approvals or denials, you can build a [ DeferredToolResults](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) object with an 

`approvals` dictionary that maps each tool call ID to a boolean, a [object (with optional](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolApproved)

`ToolApproved``override_args`), or a [object (with an optional custom](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDenied)

`ToolDenied``message` to provide to the model). You can also provide a `metadata` dictionary on `DeferredToolResults` that maps each tool call ID to a dictionary of metadata that will be available in the tool’s [attribute. This](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.tool_call_metadata)

`RunContext.tool_call_metadata``DeferredToolResults` object can then be provided to one of the agent run methods as `deferred_tool_results`, alongside the original run’s [message history](/docs/ai/core-concepts/message-history).

Here’s an example that shows how to require approval for all file deletions, and for updates of specific protected files:

The optional `metadata` parameter can attach arbitrary context to deferred tool calls, accessible in `DeferredToolRequests.metadata` keyed by `tool_call_id`.

This second agent run continues from where the first run left off, providing the tool approval results and optionally a new `user_prompt` to give the model additional instructions alongside the deferred results.

*(This example is complete, it can be run “as is”)*

When the result of a tool call cannot be generated inside the same agent run in which it was called, the tool is considered to be external. Examples of external tools are client-side tools implemented by a web or app frontend, and slow tasks that are passed off to a background worker or external service instead of keeping the agent process running.

If whether a tool call should be executed externally depends on the tool call arguments, the agent [run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) (e.g. [dependencies](/docs/ai/core-concepts/dependencies) or message history), or how long the task is expected to take, you can define a tool function and conditionally raise the [ CallDeferred](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.CallDeferred) exception. Before raising the exception, the tool function would typically schedule some background task and pass along the 

[so that the result can be matched to the deferred tool call later.](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.tool_call_id)

`RunContext.tool_call_id`If a tool is always executed externally and its definition is provided to your code along with a JSON schema for its arguments, you can use an [ ExternalToolset](/docs/ai/tools-toolsets/toolsets#external-toolset). If the external tools are known up front and you don’t have the arguments JSON schema handy, you can also define a tool function with the appropriate signature that does nothing but raise the 

[exception.](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.CallDeferred)

`CallDeferred`When the model calls an external tool, the agent run will end with a [ DeferredToolRequests](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) output object with a 

`calls` list holding [containing the tool name, validated arguments, and a unique tool call ID.](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)

`ToolCallPart`sOnce the tool call results are ready, you can build a [ DeferredToolResults](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) object with a 

`calls` dictionary that maps each tool call ID to an arbitrary value to be returned to the model, a [object, or a](/docs/ai/tools-toolsets/tools-advanced#advanced-tool-returns)

`ToolReturn`[exception in case the tool call failed and the model should](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry)

`ModelRetry`[try again](/docs/ai/tools-toolsets/tools-advanced#tool-retries). This

`DeferredToolResults` object can then be provided to one of the agent run methods as `deferred_tool_results`, alongside the original run’s [message history](/docs/ai/core-concepts/message-history).

Here’s an example that shows how to move a task that takes a while to complete to the background and return the result to the model once the task is complete:

Generate a task ID that can be tracked independently of the tool call ID.

The optional `metadata` parameter passes the `task_id` so it can be matched with results later, accessible in `DeferredToolRequests.metadata` keyed by `tool_call_id`.

In reality, this would typically happen in a separate process that polls for the task status or is notified when all pending tasks are complete.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

- [Function Tools](/docs/ai/tools-toolsets/tools)- Basic tool concepts and registration
- [Advanced Tool Features](/docs/ai/tools-toolsets/tools-advanced)- Custom schemas, dynamic tools, and execution details
- [Toolsets](/docs/ai/tools-toolsets/toolsets)- Managing collections of tools, including- `ExternalToolset`for external tools
- [Message History](/docs/ai/core-concepts/message-history)- Understanding how to work with message history for deferred tools

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/deferred-tools
