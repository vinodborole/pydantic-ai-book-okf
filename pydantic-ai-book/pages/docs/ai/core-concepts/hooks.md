---
type: Web Page
title: Hooks | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/hooks
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Hooks

Hooks let you intercept and modify agent behavior at every stage of a run — model requests, tool calls, streaming events — using simple decorators or constructor arguments. No subclassing needed.

The [`Hooks`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) capability is the recommended way to add [lifecycle hooks](/docs/ai/capabilities/custom/#hooking-into-the-lifecycle) for application-level concerns like logging, metrics, and lightweight validation. For reusable capabilities that combine hooks with tools, instructions, or model settings, subclass [`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) instead — see [Building custom capabilities](/docs/ai/capabilities/custom/).

Create a [`Hooks`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) instance, register hooks via `@hooks.on.*` decorators, and pass it to your agent:

The `hooks.on` namespace provides decorator methods for every lifecycle hook. Use them as bare decorators or with parameters:

```
# Bare decorator
@hooks.on.before_model_request
async def my_hook(ctx, request_context):
    return request_context
# With parameters (timeout, tool filter)
@hooks.on.before_model_request(timeout=5.0)
async def my_timed_hook(ctx, request_context):
    return request_context
```
Multiple hooks can be registered for the same event — they fire in registration order.

You can also pass hook functions directly to the [`Hooks`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) constructor:

Both sync and async hook functions are accepted. Sync functions are automatically wrapped for async execution.

[`Hooks`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) is a capability, so it can be loaded on demand just like any other capability. This is useful for optional, user-requested behavior such as verbose request logging:

Pydantic AI skips hooks owned by a deferred `Hooks` instance until its capability is loaded.

Use on-demand hooks for optional behavior that only applies after the capability is loaded. For human-in-the-loop tool approval, pass [`requires_approval=True`](/docs/ai/tools-toolsets/deferred-tools/#human-in-the-loop-tool-approval) when registering a tool, raise [`ApprovalRequired`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ApprovalRequired) for conditional approval, or wrap a toolset with [`ApprovalRequiredToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.ApprovalRequiredToolset).

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `before_run` | `before_run=` | `before_run` | 
| `after_run` | `after_run=` | `after_run` | 
| `run` | `run=` | `wrap_run` | 
| `run_error` | `run_error=` | `on_run_error` | 

Run hooks fire once per agent run. `wrap_run` (registered via `hooks.on.run`) wraps the entire run and supports error recovery.

A [realtime session](/docs/ai/realtime/capabilities/) is a run: the same four hooks fire once around the session, with `wrap_run` recovery and `after_run` result transformation applied when the session closes.

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `before_node_run` | `before_node_run=` | `before_node_run` | 
| `after_node_run` | `after_node_run=` | `after_node_run` | 
| `node_run` | `node_run=` | `wrap_node_run` | 
| `node_run_error` | `node_run_error=` | `on_node_run_error` | 

Node hooks fire for each graph step ([`UserPromptNode`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.UserPromptNode), [`ModelRequestNode`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.ModelRequestNode), [`CallToolsNode`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.CallToolsNode)).

Node hooks fire no matter how the run is driven: [`agent.run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run), [`agent_run.next()`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun.next), and `async for node in agent_run:` over [`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter) all advance the run the same way.

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `before_model_request` | `before_model_request=` | `before_model_request` | 
| `after_model_request` | `after_model_request=` | `after_model_request` | 
| `model_request` | `model_request=` | `wrap_model_request` | 
| `model_request_error` | `model_request_error=` | `on_model_request_error` | 

Model request hooks fire around each LLM call. [`ModelRequestContext`](/docs/ai/api/models/base/#pydantic_ai.models.ModelRequestContext) bundles `model`, `messages`, `model_settings`, and `model_request_parameters`. To swap the model for a given request, set `request_context.model` to a different [`Model`](/docs/ai/api/models/base/#pydantic_ai.models.Model) instance.

To skip the model call entirely, raise [`SkipModelRequest(response)`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.SkipModelRequest) from `before_model_request` or `model_request` (wrap).

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `before_tool_validate` | `before_tool_validate=` | `before_tool_validate` | 
| `after_tool_validate` | `after_tool_validate=` | `after_tool_validate` | 
| `tool_validate` | `tool_validate=` | `wrap_tool_validate` | 
| `tool_validate_error` | `tool_validate_error=` | `on_tool_validate_error` | 

Validation hooks fire when the model’s JSON arguments are parsed and validated. All tool hooks receive `call` ([`ToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)) and `tool_def` ([`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)) parameters.

To skip validation, raise [`SkipToolValidation(args)`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.SkipToolValidation) from `before_tool_validate` or `tool_validate` (wrap).

A tool call can only be [deferred](/docs/ai/tools-toolsets/deferred-tools/) once its arguments have been validated, since whoever resolves the deferral is shown those arguments. So [`ApprovalRequired`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ApprovalRequired) and [`CallDeferred`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.CallDeferred) can be raised from `after_tool_validate` (and from `tool_validate` after its `handler()` has returned), but raising them from `before_tool_validate`, from `tool_validate` before it calls `handler()`, or from `tool_validate_error` is a [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError). To decide per tool rather than per capability, use the tool’s [`args_validator`](/docs/ai/tools-toolsets/tools-advanced/#args-validator).

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `before_tool_execute` | `before_tool_execute=` | `before_tool_execute` | 
| `after_tool_execute` | `after_tool_execute=` | `after_tool_execute` | 
| `tool_execute` | `tool_execute=` | `wrap_tool_execute` | 
| `tool_execute_error` | `tool_execute_error=` | `on_tool_execute_error` | 

Execution hooks fire when the tool function runs. `args` is always the validated `dict[str, Any]`.

To skip execution, raise [`SkipToolExecution(result)`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.SkipToolExecution) from `before_tool_execute` or `tool_execute` (wrap).

Every execution hook can [defer](/docs/ai/tools-toolsets/deferred-tools/) the call — the arguments are validated by this point — but raise `ApprovalRequired`/`CallDeferred` from `before_tool_execute` (or from `tool_execute` before it calls `handler()`). A deferral from `after_tool_execute`, or from `tool_execute` after `handler()` has returned, is accepted but happens too late to be useful: the tool function already ran, so its side effects happened and its result is discarded.

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `before_output_validate` | `before_output_validate=` | `before_output_validate` | 
| `after_output_validate` | `after_output_validate=` | `after_output_validate` | 
| `output_validate` | `output_validate=` | `wrap_output_validate` | 
| `output_validate_error` | `output_validate_error=` | `on_output_validate_error` | 

Output validation hooks fire when structured output is parsed against the output schema. They do **not** fire for plain text or image output. All output hooks receive an `output_context` ([`OutputContext`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.OutputContext)) parameter.

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `before_output_process` | `before_output_process=` | `before_output_process` | 
| `after_output_process` | `after_output_process=` | `after_output_process` | 
| `output_process` | `output_process=` | `wrap_output_process` | 
| `output_process_error` | `output_process_error=` | `on_output_process_error` | 

Output processing hooks fire when the output is processed — extracting values, calling output functions, and running output validators.

See [Output hooks](/docs/ai/capabilities/custom/#output-hooks) for the full lifecycle, signatures, and details on how output validators interact with processing hooks.

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `prepare_tools` | `prepare_tools=` | `prepare_tools` | 
| `prepare_output_tools` | `prepare_output_tools=` | `prepare_output_tools` | 

Filters or modifies tool definitions the model sees on each step.

`prepare_tools` handles **function** tools; `prepare_output_tools` handles [output tools](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput) separately, with `ctx.max_retries` reflecting the **output** retry budget. Both run as `PreparedToolset` wrappers — the result flows into the model’s request *and* `ToolManager.tools`, so filtering also blocks tool execution.

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `deferred_tool_calls` | `deferred_tool_calls=` | `handle_deferred_tool_calls` | 

Resolves [deferred tool calls](/docs/ai/tools-toolsets/deferred-tools/) (approval-required or externally-executed) inline during a run. The hook receives a [`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) and returns a [`DeferredToolResults`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) (or `None` to decline). Multiple registered hooks accumulate: each receives the still-unresolved requests and can resolve some or all of them.

For pure application-level handler registration without other hooks, the dedicated [`HandleDeferredToolCalls`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.HandleDeferredToolCalls) capability is more concise — see [Resolving deferred calls with a handler](/docs/ai/tools-toolsets/deferred-tools/#resolving-deferred-calls-with-a-handler).

| `hooks.on.` | Constructor kwarg | `AbstractCapability` method | 
|---|---|---|
| `run_event_stream` | `run_event_stream=` | `wrap_run_event_stream` | 
| `event` | `event=` | *(per-event convenience)* | 

`run_event_stream` wraps the full event stream as an async generator. `event` is a convenience — it fires for each individual event during a streamed run. Tool and model events flow through this stream, along with framework events such as [`EnqueuedMessagesEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.EnqueuedMessagesEvent) when queued messages enter run history. During a [realtime session](/docs/ai/realtime/capabilities/), both hooks also fire, and realtime-only [`RealtimeEvent`](/docs/ai/api/pydantic-ai/realtime/#pydantic_ai.realtime.RealtimeEvent) members flow through the same stream:

Tool hooks (validation and execution) support a `tools` parameter to target specific tools by name:

The `tools` parameter accepts a sequence of tool names. The hook only fires for matching tools — other tool calls pass through unaffected.

Each hook supports an optional `timeout` in seconds. If the hook exceeds the timeout, a [`HookTimeoutError`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.HookTimeoutError) is raised:

Timeouts are set via the decorator parameter (`@hooks.on.before_model_request(timeout=5.0)`) or via the constructor when using kwargs.

Wrap hooks let you surround an operation with setup/teardown logic. In the `hooks.on` namespace, wrap hooks drop the `wrap_` prefix — `hooks.on.model_request` corresponds to `wrap_model_request`:

Within a single [`Hooks`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) instance, `before_*`, `after_*`, and `on_*_error` fire in **registration order** (the order they were defined or passed to the constructor). `wrap_*` nests as middleware, with the first-registered wrapper as the outermost layer.

Across multiple capabilities, the [composition rules](/docs/ai/capabilities/custom/#composition-and-middleware-semantics) apply: `before_*` fires in capability order, `after_*` fires in reverse capability order, and `wrap_*` nests as middleware with the first capability outermost.

Hook timing also affects what is populated on [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext). Early run and node hooks can fire before the current step’s tool manager and model request parameters have been assembled. At that point `ctx.available_tool_names` can still include tool-search discoveries reconstructed from history, but `ctx.tools` and current request parameters may be empty or reflect the previous step. `before_model_request` and later model-request hooks see the request about to be sent, including the current function tools, native tools, and model settings. Tool and output hooks see the state for the call or output currently being processed.

For on-demand capabilities, `ctx.loaded_capability_ids` is derived from message history before each model request, so a capability loaded during a step appears from the *next* step onwards — the same step that first carries its instructions to the model, and therefore the first on which its tools can be called. Function tools, native tools, and model settings from the loaded capability appear on that request too, and hooks owned by the capability run for hook points reached from then on. A hook that looks for a capability in the very turn it was loaded will not find it.

Error hooks (`*_error` in the `hooks.on` namespace, `on_*_error` on `AbstractCapability`) use **raise-to-propagate, return-to-recover** semantics:

- **Raise the original error** — propagates unchanged*(default)*
- **Raise a different exception** — transforms the error
- **Return a result** — suppresses the error

See [Error hooks](/docs/ai/capabilities/custom/#error-hooks) for the full pattern and recovery types.

Hooks can raise [`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) to ask the model to try again with a custom message — the same exception used in [tool functions](/docs/ai/tools-toolsets/tools-advanced/#tool-retries) and output validators.

**Model request hooks** (`after_model_request`, `wrap_model_request`, `on_model_request_error`):

- The retry message is sent back to the model as a [`RetryPromptPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.RetryPromptPart)
- `after_model_request` : the original response is preserved in message history so the model can see what it said
- `wrap_model_request` : the response is preserved only if the handler was called
- Retries count against the output side of the agent’s retry budget

**Tool hooks** (`before/after_tool_validate`, `before/after_tool_execute`, `wrap_tool_execute`, `on_tool_execute_error`):

- Converted to tool retry prompts, same as when a tool function raises `ModelRetry`
- Retries count against the tool’s `max_retries` limit

**Output hooks** (`before/after_output_validate`, `before/after_output_process`, `wrap_output_process`, `on_output_process_error`):

- Converted to retry prompts, same as when an output function raises `ModelRetry`
- For tool output, retries count against the tool’s `max_retries` limit
- For text output, retries count against the output side of the agent’s retry budget

[`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) from `wrap_model_request`, `wrap_tool_execute`, or `wrap_output_process` is control flow and bypasses the corresponding `on_*_error` hook. [`ToolFailed`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ToolFailed) is control flow only at the tool boundary, so it bypasses `on_tool_execute_error`. From model-request and output-process hooks, `ToolFailed` is an ordinary exception and is passed to `on_model_request_error` or `on_output_process_error`.

Tool validation and execution hooks can also raise [`ToolFailed`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ToolFailed) to report a failed tool result without consuming the tool’s retry budget. This has the same model-visible outcome and retry-budget behavior as raising `ToolFailed` from the tool function itself, and is useful when an error hook converts a third-party exception into a failure the model can see.

By default, any exception other than `ModelRetry` or `ToolFailed` raised inside a tool escapes the
tool boundary and aborts the entire run. A tool-execution hook lets you intercept these in one
place — without editing every tool — and choose how each surfaces to the model. The distinction is
the semantic one between [requesting a retry](/docs/ai/tools-toolsets/tools-advanced/#tool-retries) and
[reporting a failure](/docs/ai/tools-toolsets/tools-advanced/#tool-failed):

- raise `ModelRetry` for**transient** errors, where the same call might succeed if tried again;
- raise `ToolFailed` for**definitive** failures, where retrying won’t help and the model should
see the result and adapt (choose another approach, tell the user, etc.).

The hook below makes that call based on an upstream status code — the per-error analogue of the
MCP [`tool_error_behavior`](/docs/ai/mcp/client/#tool-errors) setting:

Because the failure was raised as `ToolFailed` rather than `ModelRetry`, the model receives it as a
[`ToolReturnPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolReturnPart) with `outcome='failed'` and decides what to
do next, instead of burning a retry on a call that can’t succeed.

| Use [`Hooks`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) | Use [`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) | 
|---|---|
| Application-level hooks (logging, metrics) | Reusable, packaged capabilities | 
| Quick one-off interceptors | Combined tools + hooks + instructions + settings | 
| No configuration state needed | Complex per-run state management | 
| Single-file scripts | Multi-agent shared behavior |

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/hooks
