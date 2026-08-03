---
type: Web Page
title: Temporal | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/durable_execution/temporal
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Temporal

[Temporal](https://temporal.io) is a popular [durable execution](https://docs.temporal.io/evaluate/understanding-temporal#durable-execution) platform that’s natively supported by Pydantic AI.

In Temporal’s durable execution implementation, a program that crashes or encounters an exception while interacting with a model or API will retry until it can successfully complete.

Temporal relies primarily on a replay mechanism to recover from failures. As the program makes progress, Temporal saves key inputs and decisions, allowing a re-started program to pick up right where it left off.

The key to making this work is to separate the application’s repeatable (deterministic) and non-repeatable (non-deterministic) parts:

1. Deterministic pieces, termed [**workflows**](https://docs.temporal.io/workflow-definition) , execute the same way when re-run with the same inputs.
2. Non-deterministic pieces, termed [**activities**](https://docs.temporal.io/activities) , can run arbitrary code, performing I/O and any other operations.

Workflow code can run for extended periods and, if interrupted, resume exactly where it left off.
Critically, workflow code generally *cannot* include any kind of I/O, over the network, disk, etc.
Activity code faces no restrictions on I/O or external interactions, but if an activity fails part-way through it is restarted from the beginning.

In the case of Pydantic AI agents, integration with Temporal means that [model requests](/docs/ai/models/overview), [tool calls](/docs/ai/tools-toolsets/tools) that may require I/O, and [MCP server communication](/docs/ai/mcp/client) all need to be offloaded to Temporal activities due to their I/O requirements, while the logic that coordinates them (i.e. the agent run) lives in the workflow. Code that handles a scheduled job or web request can then execute the workflow, which will in turn execute the activities as needed.

The diagram below shows the overall architecture of an agentic application in Temporal. The Temporal Server is responsible for tracking program execution and making sure the associated state is preserved reliably (i.e., stored to an internal database, and possibly replicated across cloud regions). Temporal Server manages data in encrypted form, so all data processing occurs on the Worker, which runs the workflow and activities.

```
            +---------------------+
            |   Temporal Server   |      (Stores workflow state,
            +---------------------+       schedules activities,
                     ^                    persists progress)
                     |
        Save state,  |   Schedule Tasks,
        progress,    |   load state on resume
        timeouts     |
                     |
+------------------------------------------------------+
|                      Worker                          |
|   +----------------------------------------------+   |
|   |              Workflow Code                   |   |
|   |       (Agent Run Loop)                       |   |
|   +----------------------------------------------+   |
|          |          |                |               |
|          v          v                v               |
|   +-----------+ +------------+ +-------------+       |
|   | Activity  | | Activity   | |  Activity   |       |
|   | (Tool)    | | (MCP Tool) | | (Model API) |       |
|   +-----------+ +------------+ +-------------+       |
|         |           |                |               |
+------------------------------------------------------+
          |           |                |
          v           v                v
      [External APIs, services, databases, etc.]
```
See the [Temporal documentation](https://docs.temporal.io/evaluate/understanding-temporal#temporal-application-the-building-blocks) for more information.

To make a run durable, call `agent.run()` inside a Temporal workflow executed on a worker; outside one, the agent runs as a normal, non-durable agent.

Add durable execution to any [`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) by attaching the `TemporalDurability`[capability](/docs/ai/capabilities/overview). The agent stays a normal `Agent` everywhere — outside a workflow it behaves transparently, and inside a workflow the capability routes model requests, tool calls, and MCP server communication through Temporal activities.

Here is a simple but complete example of attaching durable execution to an agent, creating a Temporal workflow with durable execution logic, connecting to a Temporal server, and running the workflow from non-durable code. All it requires is to install Pydantic AI with the `temporal` optional group:

Or if you’re using the slim package, you can install it with the `temporal` optional group:

You’ll also need a Temporal server to be [running locally](https://github.com/temporalio/temporal#download-and-start-temporal-server-locally):

The agent's `name` is used to uniquely identify its activities.

Attach durability via `capabilities=[...]`. The capability discovers the agent's name, model, and toolsets when bound to the agent, and registers an activity for each. Outside a workflow, the capability is transparent — the agent behaves as a normal `Agent`.

The workflow represents a deterministic piece of code that can use non-deterministic activities for operations that require I/O. Subclassing [`PydanticAIWorkflow`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.PydanticAIWorkflow) is optional but provides proper typing for the `__pydantic_ai_agents__` class variable.

List the agents used by this workflow. The [`PydanticAIPlugin`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.PydanticAIPlugin) automatically registers the activities contributed by each agent's [`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability) capability with the worker. Alternatively, if modifying the worker initialization is easier than the workflow class, you can use [`AgentPlugin`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.AgentPlugin) to register an agent's activities directly on the worker.

`agent.run()` works as usual; inside the workflow, model requests, tool calls, and MCP server communication are routed through Temporal activities.

We connect to the Temporal server which keeps track of workflow and activity execution.

This assumes the Temporal server is [running locally](https://github.com/temporalio/temporal#download-and-start-temporal-server-locally).

The [`PydanticAIPlugin`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.PydanticAIPlugin) tells Temporal to use Pydantic for serialization and deserialization, and automatically registers activities for agents listed in `__pydantic_ai_agents__`. Activity retry policies treat [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError), `PydanticUserError`, [`UnexpectedModelBehavior`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UnexpectedModelBehavior), and [`FallbackExceptionGroup`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.FallbackExceptionGroup) as non-retryable, while the worker registers `UserError`, `PydanticUserError`, [`AgentRunError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.AgentRunError), and `UnsupportedEventLoopError` as `workflow_failure_exception_types`.

We start the worker that will listen on the specified task queue and run workflows and activities. In a real world application, this might be run in a separate service.

We call on the server to execute the workflow on a worker that's listening on the specified task queue.

*(This example is complete, it can be run “as is” — you’ll need to add `asyncio.run(main())` to run `main`)*

Because the same agent works inside and outside a workflow, [`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability) composes with all other [capabilities](/docs/ai/capabilities/overview) (instrumentation, [`SetToolMetadata`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.SetToolMetadata), [`ProcessEventStream`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ProcessEventStream), etc.) without each needing a Temporal-specific wrapper variant.

Workflow code has to use the async API: `Agent.run_sync()` drives the event loop itself, which Temporal’s workflow event loop doesn’t allow, so calling it inside a workflow raises a [`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) telling you to `await agent.run()` instead. Outside a workflow, an agent with `TemporalDurability` behaves like a normal agent, so `run_sync()` works as usual.

In a real world application, the agent, workflow, and worker are typically defined separately from the code that calls for a workflow to be executed. Because Temporal workflows need to be defined at the top level of the file and the agent is needed inside the workflow and when starting the worker (to register the activities), it needs to be defined at the top level of the file as well.

For more information on how to use Temporal in Python applications, see their [Python SDK guide](https://docs.temporal.io/develop/python).

Any agent can be wrapped in a [`TemporalAgent`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalAgent) to get a durable agent variant that can be used inside a Temporal workflow. At the time of wrapping, the agent’s model and toolsets are frozen, activities are dynamically created for each, and the original model and toolsets are wrapped to call on the worker to execute the corresponding activities instead of directly performing the actions inside the workflow. The original agent can still be used as normal outside the Temporal workflow, but any changes to its model or toolsets after wrapping will not be reflected in the durable agent.

There are a few considerations specific to agents and toolsets when using Temporal for durable execution. These are important to understand to ensure that your agents and toolsets work correctly with Temporal’s workflow and activity model.

To ensure that Temporal knows what code to run when an activity fails or is interrupted and then restarted, even if your code is changed in between, each activity needs to have a name that’s stable and unique.

When [`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability) dynamically creates activities for the agent’s model requests and toolsets (specifically those that implement their own tool listing and calling, i.e. [`FunctionToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset) and [`MCPToolset`](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset)), their names are derived from the agent’s [`name`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.name) and the toolsets’ [`id`s](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.id). These fields are normally optional, but are required to be set when using Temporal. They should not be changed once the durable agent has been deployed to production as this would break active workflows.

`DynamicToolset` and toolsets contributed by [`DynamicCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.DynamicCapability) are supported. Their factory is re-resolved inside activities when tools are listed and called, so it must be deterministic given the run dependencies. Like other wrapped toolsets, every `DynamicToolset` requires an explicit `id`: pass `id=` when constructing one directly, set the `id` parameter of the [`@agent.toolset`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.toolset) decorator, or set a stable capability `id` on `DynamicCapability`. Note that with Temporal, `per_run_step=False` is not respected, as the toolset always needs to be created on-the-fly in the activity.

[Capabilities](/docs/ai/capabilities/overview) that contribute a toolset — a [`Capability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) with `tools=`, or an [`MCP`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.MCP) server running locally — derive the toolset’s `id` from the capability’s own [`id`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.id), so set `Capability(id='...', tools=[...])` or `MCP(id='...', url='...')`. (`MCP` falls back to an id derived from the server URL’s host and path when no `id` is given.) A toolset passed to a capability via `toolsets=` keeps its own `id`, which must be set on the toolset itself.

Other than that, any agent and toolset will just work!

As workflows and activities run in separate processes, any values passed between them need to be serializable. As these payloads are stored in the workflow execution event history, Temporal limits their size to 2MB.

To account for these limitations, tool functions and the [event stream handler](#streaming) running inside activities receive a limited version of the agent’s [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext), and it’s your responsibility to make sure that the [dependencies](/docs/ai/core-concepts/dependencies) object provided to `Agent.run()` can be serialized using Pydantic.

`deps` isn’t the only value that crosses into an activity: [`model_settings`](/docs/ai/core-concepts/agent#model-run-settings), the `RunContext` `metadata` and `tool_call_metadata`, and [tool metadata](#per-tool-activity-config) do too, and all need to be serializable by Pydantic. A value that isn’t raises a `UserError` naming the type it couldn’t serialize. This includes some values that are otherwise supported: pass [`model_settings['timeout']`][pydantic_ai.settings.ModelSettings.timeout] as a number of seconds rather than an `httpx.Timeout`.

Values that Pydantic AI carries across the boundary as untyped dictionaries — [tool metadata](#per-tool-activity-config), the `RunContext` `metadata` and `tool_call_metadata`, the `metadata` on [`ApprovalRequired`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ApprovalRequired) and [`CallDeferred`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.CallDeferred), and the `metadata`, `provider_details`, and `vendor_metadata` carried on messages and their parts — arrive as their JSON shapes rather than the original Python objects: a `set` or `tuple` becomes a `list`, a dataclass or Pydantic model becomes a `dict`, UTF-8-decodable `bytes` become a `str`, and non-string dictionary keys become strings. Arbitrary binary doesn’t make it onto the wire at all — encode it as base64 first. There’s no type information on the wire to restore any of this from, so keep these payloads JSON-native, or re-validate them into the type you expect (with a [`TypeAdapter`](https://docs.pydantic.dev/latest/api/type_adapter/)) on the receiving side. Every field with a declared type — the rest of each message, [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition), and [`ModelRequestParameters`](/docs/ai/api/models/base/#pydantic_ai.models.ModelRequestParameters), as well as the run context’s `usage`, `usage_limits`, `loaded_capability_ids`, and `discovered_tool_names` — round-trips faithfully.

Specifically, only the `deps`, `run_id`, `conversation_id`, `metadata`, `retries`, `tool_call_id`, `tool_name`, `tool_call_approved`, `tool_call_metadata`, `retry`, `max_retries`, `run_step`, `partial_output`, `usage`, `usage_limits`, `trace_include_content`, `instrumentation_version`, `loaded_capability_ids`, `discovered_tool_names`, and `capability_loaded` fields are available by default. `agent` and `root_capability` are re-attached from the worker’s agent instance, `tool_manager` is `None` (which is what makes `available_tool_names` fall back to `discovered_tool_names`), and `pending_messages` holds a guard that makes [`ctx.enqueue()`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.enqueue) raise inside an activity, since the activity’s recorded result is replayed without re-running your code and the enqueued messages would be dropped.

Trying to access any other field — `model`, `prompt`, `messages`, `model_settings`, `tracer`, `validation_context`, or `capabilities` — raises a `UserError` rather than returning that field’s default value, so a field that didn’t cross the boundary can’t be mistaken for real run state. A multi-modal `prompt` can carry large [`BinaryContent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryContent), and carrying it would put that content in every activity payload, creating the same Temporal 2 MB concern as `messages`.
If you need one or more of these attributes to be available inside activities, you can create a [`TemporalRunContext`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalRunContext) subclass with custom `serialize_run_context` and `deserialize_run_context` class methods and pass it as the `run_context_type` argument to [`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability). A subclass can opt in to carrying `prompt` if it knows its prompts are text-only.

The activity’s `RunContext` is rebuilt from the serialized payload, so its fields are copies: mutating them inside an activity does not affect the run. In particular, `usage` is a snapshot of the run’s usage at the time the activity was scheduled. If a tool [delegates to another agent](/docs/ai/guides/multi-agent-applications#agent-delegation) with `usage=ctx.usage`, the delegate’s tokens and requests stay behind in the activity: they’re missing from the parent run’s [`result.usage`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult.usage) and are never charged against its [usage limits](/docs/ai/core-concepts/agent#usage-limits). To account for delegate usage, carry it yourself: return the delegate’s [`result.usage`](/docs/ai/api/pydantic-ai/run/#pydantic_ai.run.AgentRunResult.usage) from the tool, or record it in an external store your `deps` can reach. Temporal loses these mutations unconditionally. [DBOS](/docs/ai/capabilities/durable_execution/dbos) and [Prefect](/docs/ai/capabilities/durable_execution/prefect) pass the live `RunContext` into their in-process durable units, so mutations do accrue while a step or task body actually runs — but they’re lost there too whenever the body doesn’t run because its recorded result is replayed (DBOS workflow recovery) or reused (a Prefect task cache hit), which makes the same code account differently from one run to the next. Don’t rely on the in-process engines’ behavior; a return channel that works for all three is under discussion in [pydantic-ai#6886](https://github.com/pydantic/pydantic-ai/issues/6886).

A tool’s [`prepare`](/docs/ai/tools-toolsets/tools-advanced#tool-prepare) function is not affected by these limitations: for tools in a [`FunctionToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset) (including those defined on the agent itself), it runs in workflow code with the complete `RunContext`, once per run step like outside a workflow. The tool definition it returns is sent to the tool-call activity, which uses it as-is, so the tool the model saw is the tool that runs, down to its [`timeout`](/docs/ai/tools-toolsets/tools-advanced#tool-timeout). Tools from a `DynamicToolset` are the exception: as the toolset is re-resolved inside activities, their `prepare` functions run there as well and see the limited `RunContext`.

`Agent.run_stream()`, `Agent.run_stream_events()`, and [`Agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter) work inside a Temporal workflow, but their events are buffered rather than delivered in real time. The model stream runs inside the durable activity, and its events are replayed to the workflow after the activity completes.

For handlers with I/O side effects, pass `event_stream_handler=` to [`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability). Model events are delivered live inside each model-request activity, while each tool event is delivered in its own event-handler activity. As with any Temporal activity, a handler may run more than once if an activity retries, so keep its side effects idempotent.

Alternatively, register [`ProcessEventStream`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ProcessEventStream). Its handler runs in workflow code and must be deterministic because it re-runs on workflow replay. Tool and final-output events arrive live, while the real captured model events are replayed after each model request completes. For examples, see the [streaming docs](/docs/ai/core-concepts/agent#streaming-all-events).

A durability `event_stream_handler=` and a separately registered `ProcessEventStream` are two distinct handlers, and each fires once. The durability handler receives live events inside the durable activity, while `ProcessEventStream` sees the buffered replay in workflow code.

A per-run handler passed to `Agent.run(event_stream_handler=...)` also runs workflow-side against replayed model events.

As the streaming model request activity, workflow, and workflow execution call all take place in separate processes, passing data between them requires some care:

- To get data from the workflow call site or workflow to the event stream handler, you can use a [dependencies object](#agent-run-context-and-dependencies) .
- To get data from the event stream handler to the workflow, workflow call site, or a frontend, you need to use an external system that the event stream handler can write to and the event consumer can read from, like a message queue. You can use the dependency object to make sure the same connection string or other unique ID is available in all the places that need it.

Because the model stream is consumed inside the activity, cancelling it from the workflow side (e.g. with [`AgentStream.cancel()`](/docs/ai/api/pydantic-ai/result/#pydantic_ai.result.AgentStream.cancel)) is not available across the durable boundary. To stop an in-flight model request, cancel the Temporal workflow: the cancellation is delivered to the activity (via its heartbeats), which cancels any server-side job before the activity completes.

`Agent.run_stream_sync()` is not for workflow code: it requires no running event loop and wraps `run_stream()`. Under [`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability), use the buffered async streaming APIs above or `Agent.run()` with an event stream handler. Outside a workflow, an agent with `TemporalDurability` behaves like a normal agent, so `run_stream_sync()` works as usual. (Wrapper `TemporalAgent` forbids `run_stream` inside workflows — use `run` + event stream handler there.)

Some providers can pause a model turn mid-flight (Anthropic `pause_turn`) or run it as a server-side job that’s polled until it’s ready ([OpenAI background mode](/docs/ai/models/openai#background-mode)). Pydantic AI transparently continues such a suspended turn until it completes. Each segment runs in a separate model request activity, while the workflow checkpoints the suspended [`ModelResponse`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) and its background job ID between segments. The final response is merged and usage is recorded once. A [`message_history`](/docs/ai/core-concepts/message-history) ending in a suspended response resumes with that response passed to the first activity.

This has a few operational implications:

- **Timeouts and heartbeats** : size`start_to_close_timeout` and`heartbeat_timeout` for one provider round trip. Model request activities are given a default`heartbeat_timeout` of 30 seconds; see[Activity Configuration](#activity-configuration) for how heartbeating works across the other activities.
- **Retries and waits** : a failed segment retries independently. Delays between background polls use durable Temporal timers and do not consume activity wall-clock time.
- **Cancellation** : if an error abandons a suspended job, its provider teardown runs in a dedicated cancellation activity.
- **Payload size** : whenever[streaming](#streaming) is used — an`event_stream_handler` , a`ProcessEventStream` capability, or a per-run`event_stream_handler` — each segment’s buffered events are shipped back to the workflow and must fit within Temporal’s 2MB payload limit.

`Agent.run(model=...)` normally supports both model strings (like `'openai:gpt-5.6-sol'`) and model instances. Under Temporal, a model instance can’t be serialized for the replay mechanism, and rebuilding one from its `model_id` string would build a *different* model — the same model name on whatever provider the worker’s environment implies, so the request would go to another endpoint with other credentials. An instance that isn’t registered ahead of time is therefore rejected with a `UserError`. There are two ways to use a specific instance inside a workflow:

- pre-register it by passing a `models` dict to[`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability) , then reference it by name or by passing the registered instance directly to`agent.run(model=...)` ;
- pass a model-name string and build the instance on the worker with a [`ResolveModelId`](/docs/ai/capabilities/resolve-model-id) capability — the right choice when the model depends on the run’s`deps` , e.g. per-user credentials.

Model-name strings themselves never need registering. The agent’s own model, set at construction, is always available as the default; the agent must have a model set when it’s created.

Model strings work as expected. To customize how a model string is built — a custom provider, API keys injected from configuration, per-user credentials carried on the run’s `deps` — add a [`ResolveModelId`](/docs/ai/capabilities/resolve-model-id) capability before `TemporalDurability`: it gets first crack at every string, both at run setup and when the model is rebuilt inside the activity, where the resolver runs again with the run’s actual `deps`. Since the resolver re-runs on the worker, it must be deterministic for a given `(model_id, deps)` and must not perform external I/O — carry credentials on `deps` or close over configuration loaded at startup.

Here’s an example showing how to pre-register and use multiple models:

Additional toolsets can be passed per run via `agent.run(toolsets=...)`, but only toolsets that don’t need durable wrapping are supported: non-executing toolsets like [`ExternalToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.ExternalToolset), whose tools are executed outside the agent run, and [`FunctionToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset)s whose tools all opt out of activity wrapping with [`metadata={'temporal': False}`](#per-tool-activity-config). Other executing toolsets ([`FunctionToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset) and [`MCPToolset`](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset)) and dynamic toolsets must be set when constructing the agent so their activities can be registered with the worker before the workflow runs; passing them at runtime raises a `UserError`.

Temporal activity configuration, like timeouts and retry policies, can be customized by passing [`temporalio.workflow.ActivityConfig`](https://python.temporal.io/temporalio.workflow._activities.ActivityConfig.html) objects to the [`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability) constructor:

- `activity_config` : The base Temporal activity config to use for all activities. If no config is provided, a`start_to_close_timeout` of 60 seconds is used.
- `model_activity_config` : The Temporal activity config to use for model request activities. This is merged with the base activity config.
- `event_stream_handler_activity_config` : The Temporal activity config to use for event stream handler activities. This is merged with the base activity config.
- `toolset_activity_config` : The Temporal activity config to use for get-tools and call-tool activities for specific toolsets identified by ID. This is merged with the base activity config.

Because `ActivityConfig` is a `TypedDict`, a misspelled or misplaced key is not caught at runtime by Python and would only fail once the config is handed to Temporal inside the workflow, where the failure is retried indefinitely. Config keys are therefore checked when [`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability) is constructed, so a key Temporal doesn’t know raises a `UserError` up front.

Per-tool activity config lives on the tool itself — see [Per-tool activity config](#per-tool-activity-config) below.

Every activity Pydantic AI registers heartbeats in the background while it runs, so that a long-but-healthy activity isn’t mistaken for a crashed worker and so that cancelling the Temporal workflow can be delivered to it — see [Streaming](#streaming) for how that stops an in-flight model request. Only model request activities get a `heartbeat_timeout` by default, of 30 seconds; setting one on any other activity is up to you.

Per-tool activity config lives on the tool’s [`metadata`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.tool) field — [`TemporalDurability`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.TemporalDurability) looks for a `'temporal'` key. You can set the metadata directly on the tool definition, or apply it across a selection of tools via the [`SetToolMetadata`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.SetToolMetadata) capability. See the [capabilities documentation](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.SetToolMetadata) for the full selector vocabulary.

Inline: declare the activity config alongside the tool definition. Per-tool config merges on top of the toolset and base configs.

Set `'temporal': False` to skip activity wrapping entirely (only valid for `async` tools — sync tools always need an activity since threads aren't deterministic).

Selector-based: [`SetToolMetadata`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.SetToolMetadata) applies the same metadata across a selection of tools (`'all'`, a name list, a dict, or a callable).

On top of the automatic retries for request failures that Temporal will perform, Pydantic AI and various provider API clients also have their own request retry logic. Enabling these at the same time may cause the request to be retried more often than expected, with improper `Retry-After` handling.

When using Temporal, it’s recommended to not use [HTTP Request Retries](/docs/ai/models/http-request-retries) and to turn off your provider API client’s own retry logic, for example by setting `max_retries=0` on a [custom `OpenAIProvider` API client](/docs/ai/models/openai#custom-openai-client).

You can customize Temporal’s retry policy using [activity configuration](#activity-configuration).

Temporal generates telemetry events and metrics for each workflow and activity execution, and Pydantic AI generates events for each agent run, model request and tool call. These can be sent to [Pydantic Logfire](/docs/ai/integrations/logfire) to get a complete picture of what’s happening in your application.

To use Logfire with Temporal, you need to pass a [`LogfirePlugin`](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.temporal.LogfirePlugin) object to Temporal’s `Client.connect()`:

By default, the `LogfirePlugin` will instrument Temporal (including metrics) and Pydantic AI and send all data to Logfire. If your application already called `logfire.configure()` itself, the plugin keeps that configuration instead of replacing it, so your scrubbing options, exporters, sampling, and console settings are left alone. To customize Logfire configuration and instrumentation, you can pass a `setup_logfire` function to the `LogfirePlugin` constructor and return a custom `Logfire` instance (i.e. the result of `logfire.configure()`). To disable sending Temporal metrics to Logfire, you can pass `metrics=False` to the `LogfirePlugin` constructor.

When `logfire.info` is used inside an activity and the `pandas` package is among your project’s dependencies, you may encounter the following error which seems to be the result of an import race condition:

```
AttributeError: partially initialized module 'pandas' has no attribute '_pandas_parser_CAPI' (most likely due to a circular import)
```
To fix this, you can use the [`temporalio.workflow.unsafe.imports_passed_through()`](https://python.temporal.io/temporalio.workflow._sandbox.unsafe.html#imports_passed_through) context manager to proactively import the package and not have it be reloaded in the workflow sandbox:

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/durable_execution/temporal
