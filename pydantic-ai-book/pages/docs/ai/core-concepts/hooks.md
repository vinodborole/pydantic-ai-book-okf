---
type: Web Page
title: Hooks | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/hooks
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Hooks

Hooks let you intercept and modify agent behavior at every stage of a run — model requests, tool calls, streaming events — using simple decorators or constructor arguments. No subclassing needed.

The `Hooks` capability is the recommended way to add lifecycle hooks for application-level concerns like logging, metrics, and lightweight validation. For reusable capabilities that combine hooks with tools, instructions, or model settings, subclass `AbstractCapability` instead — see Building custom capabilities.

Create a `Hooks` instance, register hooks via `@hooks.on.*` decorators, and pass it to your agent:

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

You can also pass hook functions directly to the `Hooks` constructor:

Both sync and async hook functions are accepted. Sync functions are automatically wrapped for async execution.

`Hooks` is a capability, so it can be loaded on demand just like any other capability:

You do not need to guard hooks owned by a deferred `Hooks` instance with `ctx.capability_loaded`; Pydantic AI skips those hooks until the model calls the `load_capability` tool for that capability. Once the hook runs, `ctx.capability_loaded` is true for that hook’s owning capability. To check a different capability, inspect `ctx.loaded_capability_ids` or `ctx.available_capability_ids`.

If a hook must enforce a rule before a workflow is loaded, keep that hook in an always-available capability and inspect `ctx.loaded_capability_ids`; an on-demand hook cannot run before the model loads its own capability.

The run-scoped hooks — `before_run` and `wrap_run` — are bound at the start of the run, so a capability the model loads mid-run won’t get them for that run; they only fire when the capability is already loaded at the start (for example after resuming from message history). The capability’s per-step hooks (node, model-request, tool, output) fire from the next step onwards once it has loaded, and `after_run` fires at the end of the run if it was loaded at any point during it.

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `before_run` | `before_run=` | `before_run` | 
| `after_run` | `after_run=` | `after_run` | 
| `run` | `run=` | `wrap_run` | 
| `run_error` | `run_error=` | `on_run_error` | 

Run hooks fire once per agent run. `wrap_run` (registered via `hooks.on.run`) wraps the entire run and supports error recovery.

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `before_node_run` | `before_node_run=` | `before_node_run` | 
| `after_node_run` | `after_node_run=` | `after_node_run` | 
| `node_run` | `node_run=` | `wrap_node_run` | 
| `node_run_error` | `node_run_error=` | `on_node_run_error` | 

Node hooks fire for each graph step (`UserPromptNode`, `ModelRequestNode`, `CallToolsNode`).

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `before_model_request` | `before_model_request=` | `before_model_request` | 
| `after_model_request` | `after_model_request=` | `after_model_request` | 
| `model_request` | `model_request=` | `wrap_model_request` | 
| `model_request_error` | `model_request_error=` | `on_model_request_error` | 

Model request hooks fire around each LLM call. `ModelRequestContext` bundles `model`, `messages`, `model_settings`, and `model_request_parameters`. To swap the model for a given request, set `request_context.model` to a different `Model` instance.

To skip the model call entirely, raise `SkipModelRequest(response)` from `before_model_request` or `model_request` (wrap).

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `before_tool_validate` | `before_tool_validate=` | `before_tool_validate` | 
| `after_tool_validate` | `after_tool_validate=` | `after_tool_validate` | 
| `tool_validate` | `tool_validate=` | `wrap_tool_validate` | 
| `tool_validate_error` | `tool_validate_error=` | `on_tool_validate_error` | 

Validation hooks fire when the model’s JSON arguments are parsed and validated. All tool hooks receive `call` (`ToolCallPart`) and `tool_def` (`ToolDefinition`) parameters.

To skip validation, raise `SkipToolValidation(args)` from `before_tool_validate` or `tool_validate` (wrap).

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `before_tool_execute` | `before_tool_execute=` | `before_tool_execute` | 
| `after_tool_execute` | `after_tool_execute=` | `after_tool_execute` | 
| `tool_execute` | `tool_execute=` | `wrap_tool_execute` | 
| `tool_execute_error` | `tool_execute_error=` | `on_tool_execute_error` | 

Execution hooks fire when the tool function runs. `args` is always the validated `dict[str, Any]`.

To skip execution, raise `SkipToolExecution(result)` from `before_tool_execute` or `tool_execute` (wrap).

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `before_output_validate` | `before_output_validate=` | `before_output_validate` | 
| `after_output_validate` | `after_output_validate=` | `after_output_validate` | 
| `output_validate` | `output_validate=` | `wrap_output_validate` | 
| `output_validate_error` | `output_validate_error=` | `on_output_validate_error` | 

Output validation hooks fire when structured output is parsed against the output schema. They do **not** fire for plain text or image output. All output hooks receive an `output_context` (`OutputContext`) parameter.

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `before_output_process` | `before_output_process=` | `before_output_process` | 
| `after_output_process` | `after_output_process=` | `after_output_process` | 
| `output_process` | `output_process=` | `wrap_output_process` | 
| `output_process_error` | `output_process_error=` | `on_output_process_error` | 

Output processing hooks fire when the output is processed — extracting values, calling output functions, and running output validators.

See Output hooks for the full lifecycle, signatures, and details on how output validators interact with processing hooks.

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `prepare_tools` | `prepare_tools=` | `prepare_tools` | 
| `prepare_output_tools` | `prepare_output_tools=` | `prepare_output_tools` | 

Filters or modifies tool definitions the model sees on each step.

`prepare_tools` handles **function** tools; `prepare_output_tools` handles output tools separately, with `ctx.max_retries` reflecting the **output** retry budget. Both run as `PreparedToolset` wrappers — the result flows into the model’s request *and* `ToolManager.tools`, so filtering also blocks tool execution.

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `deferred_tool_calls` | `deferred_tool_calls=` | `handle_deferred_tool_calls` | 

Resolves deferred tool calls (approval-required or externally-executed) inline during a run. The hook receives a `DeferredToolRequests` and returns a `DeferredToolResults` (or `None` to decline). Multiple registered hooks accumulate: each receives the still-unresolved requests and can resolve some or all of them.

For pure application-level handler registration without other hooks, the dedicated `HandleDeferredToolCalls` capability is more concise — see Resolving deferred calls with a handler.

| `hooks.on.` | Constructor kwarg | `AbstractCapability`method | 
|---|---|---|
| `run_event_stream` | `run_event_stream=` | `wrap_run_event_stream` | 
| `event` | `event=` | (per-event convenience) | 

`run_event_stream` wraps the full event stream as an async generator. `event` is a convenience — it fires for each individual event during a streamed run:

Tool hooks (validation and execution) support a `tools` parameter to target specific tools by name:

The `tools` parameter accepts a sequence of tool names. The hook only fires for matching tools — other tool calls pass through unaffected.

Each hook supports an optional `timeout` in seconds. If the hook exceeds the timeout, a `HookTimeoutError` is raised:

Timeouts are set via the decorator parameter (`@hooks.on.before_model_request(timeout=5.0)`) or via the constructor when using kwargs.

Wrap hooks let you surround an operation with setup/teardown logic. In the `hooks.on` namespace, wrap hooks drop the `wrap_` prefix — `hooks.on.model_request` corresponds to `wrap_model_request`:

When multiple hooks are registered for the same event (either on the same `Hooks` instance or across multiple capabilities):

- `before_*`
- `after_*`
- `wrap_*`

Hook timing also affects what is populated on `RunContext`. Early run and node hooks can fire before the current step’s tool manager and model request parameters have been assembled. At that point `ctx.available_tool_names` can still include tool-search discoveries reconstructed from history, but `ctx.tools` and current request parameters may be empty or reflect the previous step. `before_model_request` and later model-request hooks see the request about to be sent, including the current function tools, native tools, and model settings. Tool and output hooks see the state for the call or output currently being processed.

For on-demand capabilities, `ctx.loaded_capability_ids` updates as soon as the `load_capability` tool runs. Function tools, native tools, and model settings from the loaded capability appear on the next model request, while hooks owned by that capability can only run for hook points reached after the capability has loaded.

See Composition and middleware semantics for details on how hooks from multiple capabilities interact.

Error hooks (`*_error` in the `hooks.on` namespace, `on_*_error` on `AbstractCapability`) use **raise-to-propagate, return-to-recover** semantics:

- **Raise the original error**— propagates unchanged- *(default)*
- **Raise a different exception**— transforms the error
- **Return a result**— suppresses the error

See Error hooks for the full pattern and recovery types.

Hooks can raise `ModelRetry` to ask the model to try again with a custom message — the same exception used in tool functions and output validators.

**Model request hooks** (`after_model_request`, `wrap_model_request`, `on_model_request_error`):

- The retry message is sent back to the model as a `RetryPromptPart`
- `after_model_request`: the original response is preserved in message history so the model can see what it said
- `wrap_model_request`: the response is preserved only if the handler was called
- Retries count against the output side of the agent’s retry budget

**Tool hooks** (`before/after_tool_validate`, `before/after_tool_execute`, `wrap_tool_execute`, `on_tool_execute_error`):

- Converted to tool retry prompts, same as when a tool function raises `ModelRetry`
- Retries count against the tool’s `max_retries`limit

**Output hooks** (`before/after_output_validate`, `before/after_output_process`, `wrap_output_process`, `on_output_process_error`):

- Converted to retry prompts, same as when an output function raises `ModelRetry`
- For tool output, retries count against the tool’s `max_retries`limit
- For text output, retries count against the output side of the agent’s retry budget

`ModelRetry` from `wrap_model_request`, `wrap_tool_execute`, and `wrap_output_process` is treated as control flow — it bypasses the corresponding `on_*_error` hook.

| Use `Hooks` | Use `AbstractCapability` | 
|---|---|
| Application-level hooks (logging, metrics) | Reusable, packaged capabilities | 
| Quick one-off interceptors | Combined tools + hooks + instructions + settings | 
| No configuration state needed | Complex per-run state management | 
| Single-file scripts | Multi-agent shared behavior |

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/hooks
