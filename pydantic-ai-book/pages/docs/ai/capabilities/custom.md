---
type: Web Page
title: Building Custom Capabilities | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/custom
timestamp: '2026-08-10T07:48:56.025339+00:00'
---

# Building Custom Capabilities

To build your own [capability](/docs/ai/capabilities/overview), subclass [`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) and override the methods you need. There are two categories: **configuration methods** that are called at agent construction — and re-run at run setup on the replacement instance when [`for_run`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_run) returns one (see [Per-run state isolation](#per-run-state-isolation)); [`get_wrapper_toolset`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_wrapper_toolset) is always called per-run — and **lifecycle hooks** that fire during each run.

Custom capability classes can be plain classes or dataclasses. The shared metadata attributes — [`id`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.id), [`description`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.description), and [`defer_loading`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.defer_loading) — are optional declarations on the capability object for always-available capabilities. If `id` is omitted there, Pydantic AI derives a run-local id from the class name and disambiguates duplicates within the run. Deferred capabilities require an explicit stable `id`.

Use a dataclass when you want generated constructor parameters for your own configuration fields, or for the shared metadata fields:

If you define a custom `__init__`, set only the metadata you want to expose. There is no `super().__init__()` or `__post_init__()` requirement:

When [`defer_loading=True`](/docs/ai/capabilities/on-demand), provide a stable explicit `id`; history replay depends on it, and Pydantic AI rejects deferred capabilities without one. For always-available capabilities, omitting `id` still derives a run-local id from the class name.

[`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) is generic in the agent’s dependency type — `AbstractCapability[MyDeps]` means the capability’s hooks receive a `RunContext[MyDeps]`. Use `AbstractCapability[Any]` when the capability works with any dependency type, or a specific type when it needs to access dependency fields:

A capability that provides tools returns a [toolset](/docs/ai/tools-toolsets/toolsets) from [`get_toolset`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_toolset). This can be a pre-built [`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset) instance, or a callable that receives [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and returns one dynamically:

For [native tools](/docs/ai/tools-toolsets/native-tools), override [`get_native_tools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_native_tools) to return a sequence of [`AgentNativeTool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.AgentNativeTool) instances (which includes both [`AbstractNativeTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.AbstractNativeTool) objects and callables that receive [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)).

[`get_wrapper_toolset`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_wrapper_toolset) lets a capability wrap the agent’s entire assembled toolset with a [`WrapperToolset`](/docs/ai/tools-toolsets/toolsets#changing-tool-execution). This is more powerful than providing tools — it can intercept tool execution, add logging, or apply cross-cutting behavior.

The wrapper receives the combined non-output toolset (after the [`prepare_tools`](#tool-preparation) hook has wrapped it). Output tools are added separately and are not affected.

[`get_instructions`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_instructions) adds [instructions](/docs/ai/core-concepts/agent#instructions) to the agent. Since it’s called once at agent construction, return a callable if you need dynamic values:

Instructions can also use [template strings](/docs/ai/core-concepts/agent-spec#template-strings) ([`TemplateStr('Hello {{name}}')`](/docs/ai/api/pydantic-ai/template/#pydantic_ai.template.TemplateStr)) for Handlebars-style templates rendered against the agent’s [dependencies](/docs/ai/core-concepts/dependencies). In Python code, a callable with [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) is generally preferred for IDE autocomplete.

[`get_model_settings`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model_settings) returns [model settings](/docs/ai/core-concepts/agent#model-run-settings) as a dict or a callable for per-step settings.

When model settings need to vary per step — for example, enabling thinking only on retry, or forcing a specific [`tool_choice`](/docs/ai/tools-toolsets/tools-advanced#dynamic-tool-choice-via-capabilities) until a tool has been called — return a callable:

The callable receives a [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) where `ctx.model_settings` contains the merged result of all layers resolved before this capability (model defaults and agent-level settings).

Override [`get_model()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model) when model selection is one part of a larger custom capability. Return a [`Model`](/docs/ai/api/models/base/#pydantic_ai.models.Model), a model ID string, or a sync/async callable taking [`ModelSelectionContext`](/docs/ai/api/models/base/#pydantic_ai.models.ModelSelectionContext). This example chooses a model from dependencies on every request step:

[`get_model()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model) is a synchronous configuration method, but the [`ModelSelector`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ModelSelector) it returns may be synchronous or asynchronous. [`ModelSelectionContext`](/docs/ai/api/models/base/#pydantic_ai.models.ModelSelectionContext) is separate from [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) because a complete run context requires the model currently being selected. It includes dependencies, the request step, message history, and usage. Keep `get_model()` itself cheap; perform I/O in an async selector.

A model or model ID returned directly from `get_model()` is resolved once per run. A selector returned from `get_model()` is evaluated before every logical model request step.

A capability’s model slots in below a call-site `run(model=...)` argument and a run-level `spec=` model, and above the agent constructor’s model. From highest to lowest priority:

`run()`/`iter()` argument › run `spec=` model › capability `get_model()` › agent constructor.

An [`override(model=...)`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.override) still wins over all of these. An explicit model skips capability selection entirely.

Later model contributions override earlier ones. If [`for_run()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_run) leaves the capability unchanged, its bootstrap selection is reused on step one; if it returns a replacement with a different selector, that selector makes a new step-one selection.

Fallback is complementary to selection: return a configured [`FallbackModel`](/docs/ai/api/models/fallback/#pydantic_ai.models.fallback.FallbackModel) when request failures should be retried on another model.

Override [`resolve_model_id()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.resolve_model_id) when an application-specific string needs custom provider construction, credentials, or registry lookup. Unlike model selection, resolution is first-wins: capabilities are tried in order, and normal [`infer_model()`](/docs/ai/api/models/base/#pydantic_ai.models.infer_model) behavior is used only if every resolver returns `None`.

The constructor ID remains a string through [`for_agent()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_agent), so a bound capability can install a resolver without default inference first constructing a provider with different configuration or credentials.

Resolution results are cached by model ID and resolver tree for the duration of one run. If a per-step selector returns the same string again, Pydantic AI reuses the same model, provider, and client rather than invoking the resolver again. To deliberately resolve differently on a later step, select a different ID or return a [`Model`](/docs/ai/api/models/base/#pydantic_ai.models.Model) instance directly from the selector.

Bootstrap resolution uses the capability tree after `for_agent()` binding but before `for_run()`, because resolving the first model is what makes a full [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) possible. If `for_run()` returns a replacement capability, strings selected for step one or later steps use that replacement’s resolver chain. When `for_run()` leaves the capability unchanged, the already-resolved bootstrap model is reused for step one.

Model selection and resolution are eager hooks, so deferred capabilities do not contribute them, even after they are loaded. Run-spec capabilities are known during bootstrap and can supply the first model. A [`CapabilityFunc`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CapabilityFunc), or another capability whose model is only introduced by `for_run()`, requires an existing bootstrap model because `for_run()` receives a full `RunContext`; it may replace that model starting with step one, but cannot bootstrap a model-less agent. If selecting a new model after loading a deferred capability would be useful for your application, please open an issue describing the desired step and continuation semantics.

Dynamic selection is not currently supported by durable execution capabilities. Durable runs need model IDs registered before execution and must recreate the same selected model during replay or cross-run resumption. Pass an explicit registered model for durable execution. Resuming a suspended provider request in a separate ordinary run likewise requires an explicit model when the previous model came from a selector.

| Method | Return type | Purpose | 
|---|---|---|
| [`get_toolset()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_toolset) | [`AgentToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AgentToolset) ` | None` | 
| [`get_native_tools()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_native_tools) | `Sequence[`[`AgentNativeTool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.AgentNativeTool)`]` | [Native tools](/docs/ai/tools-toolsets/native-tools) to register (including callables) | 
| [`get_wrapper_toolset()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_wrapper_toolset) | [`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset) ` | None` | 
| [`get_instructions()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_instructions) | [`AgentInstructions`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentInstructions) ` | None` | 
| [`get_model_settings()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model_settings) | [`AgentModelSettings`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentModelSettings) ` | None` | 
| [`get_model()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model) | [`AgentModel`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AgentModel) ` | None` | 
| [`resolve_model_id()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.resolve_model_id) | [`Model`](/docs/ai/api/models/base/#pydantic_ai.models.Model) ` | None` | 

Override [`for_agent()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_agent) when a reusable capability needs to inspect the agent it is attached to. The hook runs once during [`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) construction, after the agent’s own model, name, and toolsets are available and before capability contributions are extracted:

`for_agent()` is synchronous because it binds configuration during agent construction, before run dependencies or a lifecycle context exist. Keep it free of I/O; asynchronous run-specific setup belongs in [`for_run()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_run).

Return a new bound copy rather than mutating the original when the same capability may be attached to multiple agents. [`CombinedCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CombinedCapability) and [`WrapperCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapperCapability) propagate binding to their children, and the bound copy participates in all configuration hooks, including `get_model()` and `resolve_model_id()`.

The parameter is typed as [`AbstractAgent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent) so reusable capabilities depend only on the portable agent interface and remain compatible with custom agent implementations. Runs through a [`WrapperAgent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.WrapperAgent) are delegated to its wrapped agent, so Pydantic AI’s built-in wrappers do not rebind the capability to the outer wrapper.

`for_agent()` sees the constructor model exactly as the caller supplied it. In particular, a model ID remains a string while binding runs, so a bound capability can introduce `resolve_model_id()` without the default inference first constructing a provider with the wrong configuration or credentials. If the bound capability tree has no resolver and `defer_model_check=False`, normal model inference happens after binding.

Capabilities passed directly to [`run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run) or through a run spec are bound once for that run before bootstrap model selection. A [`CapabilityFunc`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CapabilityFunc) is itself bound before the run; because its returned value is normally an independently reusable capability, that value is also bound before its own `for_run()` is called. In contrast, a specialized run-bound capability returned by an ordinary capability’s [`for_run()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_run) is not passed through `for_agent()` again.

Binding hooks establish which capability participates in a run; lifecycle hooks then intercept the work it performs. The high-level order is:

`for_agent()` → bootstrap model selection and resolution → `for_run()` → per-step selection and preparation → model request → tool/output processing → run completion

| Phase | Capability work | What is available | 
|---|---|---|
| Agent binding | [`for_agent()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_agent) | Agent name, raw constructor model, toolsets, and other constructor configuration; no run dependencies or `RunContext` | 
| Run bootstrap | [`get_model()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model) , then[`resolve_model_id()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.resolve_model_id) if the selection is a string | Dependencies, message history, usage, and the lower-precedence model through selection/resolution contexts; no complete `RunContext` yet | 
| Run binding | [`for_run()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_run) | A complete [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) containing the bootstrap model; may return a run-scoped replacement capability | 
| Each logical model step | Post- `for_run()` model selection/resolution, model settings, tool preparation, and message preparation | The selected model is installed in `RunContext` before its settings, profile-sensitive tools, and model-specific message preparation are evaluated | 
| Model request and response | Model request, tool, output, node, and event-stream [hooks](#hooking-into-the-lifecycle) | The fully prepared request and the live run state appropriate to each hook | 
| Run completion | `after_run` ,`on_run_error` , and`wrap_run` completion | Final result or error, accumulated messages, and usage | 

If `for_run()` returns the original capability, the bootstrap model selection is reused for step one. A replacement capability can select a different model for step one. Continuation polling within one logical step remains pinned to that step’s selected model.

Capabilities can hook into five lifecycle points, each with up to four variants:

- **`before_*`** — fires before the action, can modify inputs
- **`after_*`** — fires after the action succeeds (in reverse capability order), can modify outputs
- **`wrap_*`** — full middleware control: receives a`handler` callable and decides whether/how to call it
- **`on_*_error`** — fires when the action fails (after`wrap_*` has had its chance to recover), can observe, transform, or recover from errors

| Hook | Signature | Purpose | 
|---|---|---|
| [`before_run`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.before_run) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`) -> None` | Observe-only notification that a run is starting | 
| [`after_run`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.after_run) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, result:` [`AgentRunResult`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult)`) ->` [`AgentRunResult`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) | Modify the final result | 
| [`wrap_run`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_run) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, handler:` [`WrapRunHandler`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapRunHandler)`) ->` [`AgentRunResult`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) | Wrap the entire run | 
| [`on_run_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_run_error) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, error: BaseException) ->` [`AgentRunResult`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) | Handle run errors (see [error hooks](#error-hooks) ) | 

`wrap_run` supports error recovery: if `handler()` raises and `wrap_run` catches the exception and returns a result instead, the error is suppressed and the recovery result is used. This works with both [`agent.run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run) and [`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter).

| Hook | Signature | Purpose | 
|---|---|---|
| [`before_node_run`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.before_node_run) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, node:` [`AgentNode`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AgentNode)`) ->` [`AgentNode`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AgentNode) | Observe or replace the node before execution | 
| [`after_node_run`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.after_node_run) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, node:` [`AgentNode`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AgentNode)`, result:` [`NodeResult`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.NodeResult)`) ->` [`NodeResult`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.NodeResult) | Modify the result (next node or [`End`](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.End) ) | 
| [`wrap_node_run`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_node_run) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, node:` [`AgentNode`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AgentNode)`, handler:` [`WrapNodeRunHandler`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapNodeRunHandler)`) ->` [`NodeResult`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.NodeResult) | Wrap each graph node execution | 
| [`on_node_run_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_node_run_error) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, node:` [`AgentNode`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AgentNode)`, error: Exception) ->` [`NodeResult`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.NodeResult) | Handle node errors (see [error hooks](#error-hooks) ) | 

[`wrap_node_run`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_node_run) fires for every node in the [agent graph](/docs/ai/core-concepts/agent#iterating-over-an-agents-graph) ([`UserPromptNode`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.UserPromptNode), [`ModelRequestNode`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.ModelRequestNode), [`CallToolsNode`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.CallToolsNode)). Override this to observe node transitions, add per-step logging, or modify graph progression:

Node hooks fire however the run is driven: [`agent.run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run), [`agent_run.next()`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun.next), and `async for node in agent_run:` over [`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter) all take the same path.

You can also use `wrap_node_run` to modify graph progression — for example, limiting the number of model requests per run:

See [Iterating Over an Agent’s Graph](/docs/ai/core-concepts/agent#iterating-over-an-agents-graph) for more about the agent graph and its node types.

| Hook | Signature | Purpose | 
|---|---|---|
| [`before_model_request`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.before_model_request) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, request_context:` [`ModelRequestContext`](/docs/ai/api/models/base/#pydantic_ai.models.ModelRequestContext)`) ->` [`ModelRequestContext`](/docs/ai/api/models/base/#pydantic_ai.models.ModelRequestContext) | Modify messages, settings, parameters, or model before the model call | 
| [`after_model_request`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.after_model_request) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, request_context:` [`ModelRequestContext`](/docs/ai/api/models/base/#pydantic_ai.models.ModelRequestContext)`, response:` [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse)`) ->` [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) | Modify the model’s response | 
| [`wrap_model_request`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_model_request) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, request_context:` [`ModelRequestContext`](/docs/ai/api/models/base/#pydantic_ai.models.ModelRequestContext)`, handler:` [`WrapModelRequestHandler`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapModelRequestHandler)`) ->` [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) | Wrap the model call | 
| [`on_model_request_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_model_request_error) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, request_context:` [`ModelRequestContext`](/docs/ai/api/models/base/#pydantic_ai.models.ModelRequestContext)`, error: Exception) ->` [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) | Handle model request errors (see [error hooks](#error-hooks) ) | 

[`ModelRequestContext`](/docs/ai/api/models/base/#pydantic_ai.models.ModelRequestContext) bundles `model`, `messages`, `model_settings`, and `model_request_parameters` into a single object, making the signature future-proof. To swap the model for a given request, set `request_context.model` to a different [`Model`](/docs/ai/api/models/base/#pydantic_ai.models.Model) instance.

To skip the model call entirely and provide a replacement response, raise [`SkipModelRequest(response)`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.SkipModelRequest) from `before_model_request` or `wrap_model_request`.

`before_model_request` hooks see the full `request_context.messages` list, including any [message history](/docs/ai/core-concepts/message-history) passed to `agent.run()`, and can modify it.

Tool processing has two phases: **validation** (parsing and validating the model’s JSON arguments against the tool’s schema) and **execution** (running the tool function). Each phase has its own hooks.

All tool hooks receive a `tool_def` parameter with the [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition).

**Validation hooks** — `args` is the raw `str | dict[str, Any]` from the model before validation, or the validated `dict[str, Any]` after:

| Hook | Signature | Purpose | 
|---|---|---|
| [`before_tool_validate`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.before_tool_validate) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, call:` [`ToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)`, tool_def:` [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)`, args:` [`RawToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.RawToolArgs)`) ->` [`RawToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.RawToolArgs) | Modify raw args before validation (e.g. JSON repair) | 
| [`after_tool_validate`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.after_tool_validate) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, call:` [`ToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)`, tool_def:` [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)`, args:` [`ValidatedToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ValidatedToolArgs)`) ->` [`ValidatedToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ValidatedToolArgs) | Modify validated args | 
| [`wrap_tool_validate`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_tool_validate) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, call:` [`ToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)`, tool_def:` [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)`, args:` [`RawToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.RawToolArgs)`, handler:` [`WrapToolValidateHandler`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapToolValidateHandler)`) ->` [`ValidatedToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ValidatedToolArgs) | Wrap the validation step | 
| [`on_tool_validate_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_tool_validate_error) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, call:` [`ToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)`, tool_def:` [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)`, args:` [`RawToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.RawToolArgs) `, error: ValidationError | `[` ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry)` ) ->`[` ValidatedToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ValidatedToolArgs) | 

To skip validation and provide pre-validated args, raise [`SkipToolValidation(args)`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.SkipToolValidation) from `before_tool_validate` or `wrap_tool_validate`.

A tool call can only be [deferred](/docs/ai/tools-toolsets/deferred-tools) once its arguments have been validated, since whoever resolves the deferral is shown those arguments. [`ApprovalRequired`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ApprovalRequired) and [`CallDeferred`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.CallDeferred) can therefore be raised from `after_tool_validate`, and from `wrap_tool_validate` once its `handler()` has returned; raising one from `before_tool_validate`, from `wrap_tool_validate` before it calls `handler()`, or from `on_tool_validate_error` (which only runs because validation failed) raises a [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) naming the hook. A permitted deferral behaves exactly like one from a tool’s [`args_validator`](/docs/ai/tools-toolsets/tools-advanced#args-validator): the tool isn’t executed, the retry budget is untouched, and the call joins the run’s [`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests).

`after_tool_validate` stays a reliable gate on validated arguments: it runs even when the `args_validator` or `wrap_tool_validate` already deferred the call, so rejecting there (with `ModelRetry` or [`ToolFailed`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ToolFailed)) wins over that deferral, deferring there replaces it, and the args it returns are the ones the deferred call carries.

**Execution hooks** — `args` is always the validated `dict[str, Any]`:

| Hook | Signature | Purpose | 
|---|---|---|
| [`before_tool_execute`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.before_tool_execute) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, call:` [`ToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)`, tool_def:` [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)`, args:` [`ValidatedToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ValidatedToolArgs)`) ->` [`ValidatedToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ValidatedToolArgs) | Modify args before execution | 
| [`after_tool_execute`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.after_tool_execute) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, call:` [`ToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)`, tool_def:` [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)`, args:` [`ValidatedToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ValidatedToolArgs)`, result: Any) -> Any` | Modify execution result | 
| [`wrap_tool_execute`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_tool_execute) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, call:` [`ToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)`, tool_def:` [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)`, args:` [`ValidatedToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ValidatedToolArgs)`, handler:` [`WrapToolExecuteHandler`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapToolExecuteHandler)`) -> Any` | Wrap execution | 
| [`on_tool_execute_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_tool_execute_error) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, call:` [`ToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart)`, tool_def:` [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)`, args:` [`ValidatedToolArgs`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ValidatedToolArgs)`, error: Exception) -> Any` | Handle execution errors (see [error hooks](#error-hooks) ) | 

To skip execution and provide a replacement result, raise [`SkipToolExecution(result)`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.SkipToolExecution) from `before_tool_execute` or `wrap_tool_execute`.

Any execution hook can defer the call, but raise `ApprovalRequired`/`CallDeferred` from `before_tool_execute` (or from `wrap_tool_execute` before it calls `handler()`) so the tool function doesn’t run: a deferral from `after_tool_execute`, or from `wrap_tool_execute` after `handler()` returned, is accepted but the tool has already executed, so its side effects happened and its result is discarded.

Tool validation and execution hooks can raise [`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) to request a retry, or [`ToolFailed`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ToolFailed) to report a failed tool result without retrying. See [triggering retries and tool failures](/docs/ai/core-concepts/hooks#triggering-retries-with-modelretry) for the full pattern.

Like tool processing, [output](/docs/ai/core-concepts/output) processing has two phases: **validation** (parsing the model’s raw output against the output schema) and **processing** (extracting the value and calling any [output function](/docs/ai/core-concepts/output#output-functions)). Each phase has its own hooks.

All output hooks receive an `output_context` parameter with [`OutputContext`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.OutputContext) (mode, output type, schema info, and tool call details for [tool output](/docs/ai/core-concepts/output#tool-output)).

**Validate hooks** fire only for structured output that requires parsing (prompted, native, tool, union output). They do not fire for plain text or image output. **Process hooks** fire for **all output types** including text, structured, and image output. For [tool output](/docs/ai/core-concepts/output#tool-output), only output hooks fire — tool hooks are skipped entirely.

**Validation hooks** — fire for structured output only; `output` is `str` (raw text) or `dict` (tool args):

| Hook | Signature | Purpose | 
|---|---|---|
| [`before_output_validate`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.before_output_validate) | `(ctx, *, output_context, output: RawOutput) -> RawOutput` | Modify raw output before validation (e.g. JSON repair) | 
| [`after_output_validate`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.after_output_validate) | `(ctx, *, output_context, output: Any) -> Any` | Modify validated output | 
| [`wrap_output_validate`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_output_validate) | `(ctx, *, output_context, output: RawOutput, handler) -> Any` | Wrap the validation step | 
| [`on_output_validate_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_output_validate_error) | `(ctx, *, output_context, output: RawOutput, error: ValidationError | ModelRetry) -> Any` | 

**Processing hooks** — fire for all output types; `output` is the validated/raw output. Output validators ([`@agent.output_validator`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.output_validator)) run inside the processing pipeline (within `wrap_output_process`), so `after_output_process` sees the fully validated result:

| Hook | Signature | Purpose | 
|---|---|---|
| [`before_output_process`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.before_output_process) | `(ctx, *, output_context, output: Any) -> Any` | Modify output before processing | 
| [`after_output_process`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.after_output_process) | `(ctx, *, output_context, output: Any) -> Any` | Modify processed result | 
| [`wrap_output_process`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_output_process) | `(ctx, *, output_context, output: Any, handler) -> Any` | Wrap processing | 
| [`on_output_process_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_output_process_error) | `(ctx, *, output_context, output: Any, error: Exception) -> Any` | Handle processing errors (see [error hooks](#error-hooks) ) | 

Output validate and process hooks can raise [`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) to ask the model to try again with a custom message — the same pattern used in [output functions](/docs/ai/core-concepts/output#output-functions) and [output validators](/docs/ai/core-concepts/output#output-validator-functions). See [Triggering retries with `ModelRetry`](/docs/ai/core-concepts/hooks#triggering-retries-with-modelretry) for the full pattern.

Capabilities can filter or modify which tool definitions the model sees on each step via two hooks:

- [`prepare_tools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.prepare_tools) — receives**function** tools only. Use this for filtering or modifications to tools the model can call directly.
- [`prepare_output_tools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.prepare_output_tools) — receives[output tools](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput) only, with`ctx.retry` /`ctx.max_retries` reflecting the**output** side of the agent retry budget, matching the[output hook](#output-hooks) lifecycle.

Both hooks operate at the toolset level — the result flows into both the model’s request parameters and `ToolManager.tools`, so filtering also blocks tool execution.

For simple cases, the built-in [`PrepareTools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrepareTools) / [`PrepareOutputTools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrepareOutputTools) capabilities wrap a callable without a custom subclass.

For runs with event streaming ([`run_stream_events`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream_events), [`event_stream_handler`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__), [UI event streams](/docs/ai/integrations/ui/overview)), capabilities can observe or transform the event stream:

| Hook | Signature | Purpose | 
|---|---|---|
| [`wrap_run_event_stream`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_run_event_stream) | `(ctx:` [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)`, *, stream: AsyncIterable[`[`AgentStreamEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AgentStreamEvent)`]) -> AsyncIterable[`[`AgentStreamEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AgentStreamEvent)`]` | Observe, filter, or transform streamed events | 

The hook wraps the stream where it’s produced, so it fires for every drive mode: [`agent.run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run) (which enables streaming automatically when this hook is registered), [`agent.run_stream()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream), and [`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter) — whether you advance it with `async for node in agent_run:`, with [`agent_run.next()`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRun.next), or by [streaming a node yourself](/docs/ai/core-concepts/agent#streaming-all-events). Events a capability drops or adds are reflected in what a manual `node.stream()` consumer sees, the same as for any other consumer.

When a consumer closes the event stream before exhausting it, Pydantic AI also closes each wrapper returned by `wrap_run_event_stream` if it provides an `aclose()` method. Custom wrappers should use `try`/`finally` for teardown and may safely await cleanup there, but must not yield events while handling `GeneratorExit` because the consumer has gone away.

Because a wrapper that closes its own input and a composed capability that closes every wrapper it built can both reach the same stream, `aclose()` may be called more than once. Async generators are idempotent here, so a `try`/`finally` wrapper needs nothing extra; a wrapper implementing `aclose()` by hand should make repeat calls a no-op.

Matching against [`ToolCallEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallEvent) and [`ToolResultEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolResultEvent) handles both function tool calls ([`FunctionToolCallEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.FunctionToolCallEvent) / [`FunctionToolResultEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.FunctionToolResultEvent)) and output tool calls ([`OutputToolCallEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.OutputToolCallEvent) / [`OutputToolResultEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.OutputToolResultEvent)). Match against the specific subclass when you need to treat them differently. [Deferred tool calls](/docs/ai/tools-toolsets/deferred-tools#observing-deferred-tool-calls-in-a-stream) additionally emit batch-level [`DeferredToolRequestsEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.DeferredToolRequestsEvent) / [`DeferredToolResultsEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.DeferredToolResultsEvent).

For building web UIs that transform streamed events into protocol-specific formats (like SSE), see the [UI event streams](/docs/ai/integrations/ui/overview) documentation and the [`UIEventStream`](/docs/ai/api/ui/base/#pydantic_ai.ui.UIEventStream) base class.

Each lifecycle point has an `on_*_error` hook — the error counterpart to `after_*`. While `after_*` hooks fire on success, `on_*_error` hooks fire on failure (after `wrap_*` has had its chance to recover):

```
before_X → wrap_X(handler)
  ├─ success ─────────→ after_X (modify result)
  └─ failure → on_X_error
        ├─ re-raise ──→ (error propagates, after_X not called)
        └─ recover ───→ after_X (modify recovered result)
```
Error hooks use **raise-to-propagate, return-to-recover** semantics:

- **Raise the original error** — propagates the error unchanged*(default)*
- **Raise a different exception** — transforms the error
- **Return a result** — suppresses the error and uses the returned value

| Hook | Fires when | Recovery type | 
|---|---|---|
| [`on_run_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_run_error) | Agent run fails | Return [`AgentRunResult`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult) | 
| [`on_node_run_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_node_run_error) | Graph node fails | Return next node or [`End`](/docs/ai/api/pydantic_graph/basenode/#pydantic_graph.basenode.End) | 
| [`on_model_request_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_model_request_error) | Model request fails | Return [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) | 
| [`on_tool_validate_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_tool_validate_error) | Tool validation fails | Return validated args `dict` | 
| [`on_tool_execute_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_tool_execute_error) | Tool execution fails | Return any tool result | 
| [`on_output_validate_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_output_validate_error) | Output validation fails | Return validated output | 
| [`on_output_process_error`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.on_output_process_error) | Output execution fails | Return any output result | 

With multiple capabilities, `on_*_error` hooks fire in **reverse** capability order (like `after_*`). The first capability to return a result **recovers** the error — remaining capabilities’ error hooks are not called. If a handler re-raises or raises a new exception, the next capability in the chain sees that exception.

Capabilities can resolve [deferred tool calls](/docs/ai/tools-toolsets/deferred-tools) — calls that require approval, or that are executed externally — directly from the agent run, without ending the run and waiting for a follow-up:

| Hook | Signature | Purpose | 
|---|---|---|
| [`handle_deferred_tool_calls`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.handle_deferred_tool_calls) | `(ctx: RunContext, *, requests: DeferredToolRequests) -> DeferredToolResults \| None` | Resolve some or all pending approval/external calls inline | 

Multiple capabilities can each handle a subset: dispatch accumulates results across the chain, passing only the still-unresolved requests to the next capability. Returning `None` (or a [`DeferredToolResults`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) with no entries) declines handling. Anything still unresolved bubbles up as a [`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) output for the caller to handle.

For application code that just needs to plug in a handler, use the dedicated [`HandleDeferredToolCalls`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.HandleDeferredToolCalls) capability — see [Resolving deferred calls with a handler](/docs/ai/tools-toolsets/deferred-tools#resolving-deferred-calls-with-a-handler).

[`WrapperCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.WrapperCapability) wraps another capability and delegates all methods to it — similar to [`WrapperToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.WrapperToolset) for toolsets. Subclass it to override specific methods while delegating the rest:

The built-in [`PrefixTools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrefixTools) is an example of a `WrapperCapability` — it wraps another capability and prefixes its tool names.

After construction-time [`for_agent()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_agent) binding, the resulting capability instance is shared across all runs of an agent. If your capability accumulates mutable state that should not leak between runs, override [`for_run`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.for_run) to return a fresh instance:

When `for_run` returns a new instance, the capability’s configuration is re-extracted from that replacement at run setup: [`get_instructions`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_instructions), [`get_toolset`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_toolset), [`get_native_tools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_native_tools), and [`get_model_settings`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model_settings) are re-invoked on it, and [`get_wrapper_toolset`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_wrapper_toolset), [`get_description`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_description), and all lifecycle hooks always run on it. The exception is model selection: [`get_model()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model) and bootstrap [`resolve_model_id()`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.resolve_model_id) run *before* `for_run` on the original instance, and the bootstrap selection is reused unless the replacement actually changes the model contribution — see [Model selection lifecycle and limitations](#model-selection-lifecycle-and-limitations).

Never mutate `self` inside `for_run` — return a new instance instead. When `for_run` returns the original unchanged, the configuration cached at agent construction is reused, so mutations to `self` would not be picked up.

Capabilities can be built dynamically ahead of each agent run using a function that takes the agent [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and returns a capability or `None`. This is useful when the capability — its instructions, model settings, hooks, or contributed toolset — depends on information specific to a run, like its [dependencies](/docs/ai/core-concepts/dependencies).

To register a dynamic capability, pass a function that takes [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) to the `capabilities` argument of the [`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) constructor or `agent.run()`. Sync and async functions are both supported. The function is called once per run and the returned capability replaces it for the rest of the run, so its instructions, model settings, toolsets, native tools, and hooks all flow through normally.

*(This example is complete, it can be run “as is”)*

To return more than one capability from a single factory, wrap them in a [`CombinedCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CombinedCapability).

When multiple capabilities are passed to an agent, they are composed into a single [`CombinedCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CombinedCapability) that follows **middleware semantics** — the same pattern used by web frameworks like Django and Starlette:

- **Configuration** is merged: instructions concatenate, model settings merge additively (later capabilities override earlier ones), toolsets combine, native tools collect.
- **`before_*`** hooks fire in capability order (outermost to innermost):`cap1 → cap2 → cap3` .
- **`after_*`** hooks fire in reverse order (innermost to outermost):`cap3 → cap2 → cap1` .
- **`wrap_*`** hooks nest as middleware:`cap1` wraps`cap2` wraps`cap3` wraps the actual operation. The first capability is the**outermost** layer.
- **`get_wrapper_toolset`** follows the same nesting: the first capability’s wrapper is outermost.

This means the first capability in the list has the first and last say on the operation — it sees the original input before any other capability, and it sees the final output after all inner capabilities have processed it.

By default, capabilities are composed in the order you list them. When a capability needs to be at a specific position regardless of where the user lists it, override [`get_ordering`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_ordering) to return a [`CapabilityOrdering`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CapabilityOrdering):

The available constraints are:

- **`position`** —`'outermost'` or`'innermost'` . Places the capability in a tier before (or after) all capabilities without that position. Multiple capabilities can share a tier; original list order breaks ties within it.
- **`wraps`** — list of capabilities this one wraps around (is outside of). Each entry can be a capability**type** (matches all instances via`issubclass` ) or a specific**instance** (matches by identity). Use when your capability needs to see the output of another:`CapabilityOrdering(wraps=[OtherCapability])` .
- **`wrapped_by`** — list of capabilities that wrap around this one (are outside of it). Accepts types or instances, like`wraps` . The inverse of`wraps` .
- **`requires`** — list of capability types that must be present. Raises[`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) if any are missing. Does not imply ordering.

When constraints are declared, [`CombinedCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.CombinedCapability) topologically sorts its children at construction time, preserving user-provided order as a tiebreaker.

[`Hooks`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Hooks) supports ordering via the `ordering` parameter, so you can declare ordering constraints without subclassing:

Capabilities don’t have direct access to each other. To share state between capabilities during a run, use a [`contextvars.ContextVar`](https://docs.python.org/3/library/contextvars.html#contextvars.ContextVar): one capability sets it (e.g. in `wrap_run` or `before_run`), and another reads it from its hooks. The order of capabilities in the `capabilities` list matters — the writer must come before the reader so its `before_*` hook runs first.

Test custom capabilities the same way you [test agents](/docs/ai/guides/testing) — using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) or [`FunctionModel`](/docs/ai/api/models/function/#pydantic_ai.models.function.FunctionModel). Create an agent with your capability and assert on the run result, messages, or any observable side effects of your hooks.

A guardrail is a capability that intercepts model requests or responses to enforce safety rules. Here’s one that scans model responses for potential PII and redacts it:

The `wrap_*` pattern is useful when you need to observe or time both the input and output of an operation. Here’s a capability that logs every model request and tool call:

To make a custom capability usable in [agent specs](/docs/ai/core-concepts/agent-spec), it needs a [`get_serialization_name`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_serialization_name) (defaults to the class name) and a constructor that accepts serializable arguments. The default [`from_spec`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.from_spec) implementation calls `cls(*args, **kwargs)`, so for simple dataclasses no override is needed:

Users register custom capability types via the `custom_capability_types` parameter on [`Agent.from_spec`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.from_spec) or [`Agent.from_file`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.from_file).

Override [`from_spec`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.from_spec) when the constructor takes types that can’t be represented in YAML/JSON. The spec fields should mirror the dataclass fields, but with serializable types:

In YAML this would be `- ConditionalTools: {hidden_tools: [dangerous_tool]}`. In Python code, the full constructor is available: `ConditionalTools(condition=my_check, hidden_tools=['dangerous_tool'])`.

See [Extensibility](/docs/ai/guides/extensibility) for packaging conventions and the broader extension ecosystem.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/custom
