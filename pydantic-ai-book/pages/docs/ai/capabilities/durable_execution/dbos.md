---
type: Web Page
title: DBOS | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/durable_execution/dbos
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# DBOS

[DBOS](https://www.dbos.dev/) is a lightweight [durable execution](https://docs.dbos.dev/architecture) library natively integrated with Pydantic AI.

DBOS workflows make your program **durable** by checkpointing its state in a database. If your program ever fails, when it restarts all your workflows will automatically resume from the last completed step.

- **Workflows**must be deterministic and generally cannot include I/O.
- **Steps**may perform I/O (network, disk, API calls). If a step fails, it restarts from the beginning.

Every workflow input and step output is durably stored in the system database. When workflow execution fails, whether from crashes, network issues, or server restarts, DBOS leverages these checkpoints to recover workflows from their last completed step.

DBOS **queues** provide durable, database-backed alternatives to systems like Celery or BullMQ, supporting features such as concurrency limits, rate limits, timeouts, and prioritization. See the [DBOS docs](https://docs.dbos.dev/architecture) for details.

The diagram below shows the overall architecture of an agentic application in DBOS. DBOS runs fully in-process as a library. Functions remain normal Python functions but are checkpointed into a database (Postgres or SQLite).

```
                    Clients
            (HTTP, RPC, Kafka, etc.)
                        |
                        v
+------------------------------------------------------+
|               Application Servers                    |
|                                                      |
|   +----------------------------------------------+   |
|   |        Pydantic AI + DBOS Libraries          |   |
|   |                                              |   |
|   |  [ Workflows (Agent Run Loop) ]              |   |
|   |  [ Steps (Tool, MCP, Model) ]                |   |
|   |  [ Queues ]   [ Cron Jobs ]   [ Messaging ]  |   |
|   +----------------------------------------------+   |
|                                                      |
+------------------------------------------------------+
                        |
                        v
+------------------------------------------------------+
|                      Database                        |
|   (Stores workflow and step state, schedules tasks)  |
+------------------------------------------------------+
```
See the [DBOS documentation](https://docs.dbos.dev/architecture) for more information.

Add durable execution to any [ Agent](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) by attaching the 

`DBOSDurability`[capability](/docs/ai/capabilities/overview). When the agent runs inside a DBOS workflow, the capability routes

[model requests](/docs/ai/models/overview)and

[MCP communication](/docs/ai/mcp/client)through DBOS steps. To make a run durable, call

`agent.run()` inside a `@DBOS.workflow`.The agent stays a normal `Agent` everywhere — outside a DBOS workflow the capability is transparent, and the original agent, model, and MCP server can still be used as normal.

Custom tool functions and event stream handlers registered on the agent directly or through another capability are **not automatically wrapped** by DBOS. An `event_stream_handler=` passed to `DBOSDurability` runs inside a DBOS step and receives live-streamed events.
If they involve non-deterministic behavior or perform I/O, you should explicitly decorate them with `@DBOS.step`.

Here is a simple but complete example of attaching durable execution to an agent. All it requires is to install Pydantic AI with the DBOS [open-source library](https://github.com/dbos-inc/dbos-transact-py):

Or if you’re using the slim package, you can install it with the `dbos` optional group:

This example uses SQLite. Postgres is recommended for production.

The agent's `name` is used to uniquely identify its workflows.

Attach durability via `capabilities=[...]`. The capability routes model requests and MCP communication through DBOS steps when the agent runs inside a workflow. Because DBOS workflows must be registered before `DBOS.launch()`, the agent must also be constructed before calling `DBOS.launch()`.

Wrap `agent.run()` in your own `@DBOS.workflow` to make the run durable.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Because the same agent works inside and outside a DBOS workflow, [ DBOSDurability](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.dbos.DBOSDurability) composes with all other 

[capabilities](/docs/ai/capabilities/overview)without each needing a DBOS-specific wrapper variant.

For more information on how to use DBOS in Python applications, see their [Python SDK guide](https://docs.dbos.dev/python/programming-guide).

Any agent can be wrapped in a [ DBOSAgent](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.dbos.DBOSAgent) to get a durable agent variant that routes model requests and MCP communication through DBOS steps:

Migrating to the capability means attaching `DBOSDurability` and adding the workflow decorator that `DBOSAgent` used to apply for you:

```
-dbos_agent = DBOSAgent(agent)
-result = await dbos_agent.run(prompt)
+agent = Agent(..., capabilities=[DBOSDurability()])
+
+@DBOS.workflow()
+async def answer(prompt: str) -> str:
+    result = await agent.run(prompt)
+    return result.output
```
When using DBOS with Pydantic AI agents, there are a few important considerations to ensure workflows and toolsets behave correctly.

Each agent instance must have a unique `name` so DBOS can correctly resume workflows after a failure or restart.

Each [ MCPToolset](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) must have a unique 

[, as DBOS derives its step names and per-run tool-defs cache key from it. This field is normally optional, but is required when using DBOS. It should not be changed once the durable agent has been deployed to production, as this would break active workflows.](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.id)

`id``DynamicToolset`s, including those contributed by a [ DynamicCapability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.DynamicCapability), are wrapped too and require a stable 

`id`. Tool discovery and calls run as `{name}__dynamic_toolset__{id}.get_tools` and `{name}__dynamic_toolset__{id}.call_tool` steps. The dynamic toolset is resolved and entered independently inside each step, so its I/O — including MCP communication — is checkpointed. For a `DynamicCapability`, DBOS reuses the capability resolved for the run inside those steps. The capability factory itself runs in workflow code and re-runs when a workflow recovers, so like all workflow code it must be deterministic given the run’s `deps`: construct the toolset in the factory and leave its I/O to the steps.A toolset contributed by a [capability](/docs/ai/capabilities/overview) — a [ Capability](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) with 

`tools=`, or a locally-running [server — derives its](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.MCP)

`MCP``id` from the capability’s own [, so set](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.id)

`id``Capability(id='...', tools=[...])` or `MCP(id='...', url='...')`. An `MCP` resolves its `id` in precedence order: an explicit `id=`, then a `native=MCPServerTool(...)` id, then a slug derived from the server URL’s host and path. A bare non-URL local client (e.g. `MCP(local=Path(...))`) with none of these stays id-less and must be given an explicit `id` to be used here.Function tools and event stream handlers registered on the agent directly or through another capability are not automatically wrapped by DBOS. An `event_stream_handler=` passed to `DBOSDurability` runs inside a DBOS step and receives live-streamed events. For directly registered tools and handlers, you can decide how to integrate them:

- Decorate with `@DBOS.step`if the function involves non-determinism or I/O.
- Skip the decorator if durability isn’t needed, so you avoid the extra DB checkpoint write.
- If the function needs to enqueue tasks or invoke other DBOS workflows, run it inside the agent’s main workflow (not as a step).

Other than that, any agent and toolset will just work!

DBOS checkpoints workflow inputs/outputs and step outputs into a database using [ pickle](https://docs.python.org/3/library/pickle.html). This means you need to make sure the 

[dependencies](/docs/ai/core-concepts/dependencies)object provided to

`Agent.run()` / `Agent.run_sync()`, and tool outputs can be serialized using pickle. You may also want to keep the inputs and outputs small (under ~2 MB). PostgreSQL and SQLite support up to 1 GB per field, but large objects may impact performance.`Agent.run(model=...)` supports both model strings (like `'openai:gpt-5.6-sol'`) and model instances. A model instance can’t be serialized across the step boundary, so it’s sent as its `model_id` string and rebuilt inside the step. That faithfully reproduces model-name strings and models with standard providers, but not an instance whose exact behavior depends on a custom provider, client, or settings — pre-register those by passing a `models` dict to [ DBOSDurability](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.dbos.DBOSDurability) and reference them by key (or pass the registered instance). The agent’s own model, set at construction, is always available as the default.

To customize how a model string is built — a custom provider, or per-user credentials carried on the run’s `deps` — add a [ ResolveModelId](/docs/ai/capabilities/resolve-model-id) capability before 

`DBOSDurability`: it gets first crack at every string, and the resolver runs again inside the step with the run’s actual `deps`, so it must be deterministic for a given `(model_id, deps)` and must not perform external I/O.`Agent.run_stream()` and `Agent.run_stream_events()` work inside a DBOS workflow, but their events are buffered rather than delivered in real time. The model stream runs inside the durable step, and its events are replayed to the workflow after the step completes.

For handlers with I/O side effects, pass `event_stream_handler=` to [ DBOSDurability](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.dbos.DBOSDurability). Model events are delivered live inside each model-request step, while each tool event is delivered in its own event-handler step. As with any DBOS step, a handler may run more than once if the workflow recovers before its step is checkpointed, so keep its side effects idempotent.

Alternatively, register [ ProcessEventStream](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ProcessEventStream). Its handler runs in workflow code and must be deterministic because it re-runs on workflow replay. Tool and final-output events arrive live, while the real captured model events are replayed after each model request completes. For examples, see the 

[streaming docs](/docs/ai/core-concepts/agent#streaming-all-events).

A durability `event_stream_handler=` and a separately registered `ProcessEventStream` are two distinct handlers, and each fires once. The durability handler receives live events inside the durable step, while `ProcessEventStream` sees the buffered replay in workflow code.

A per-run handler passed to `Agent.run(event_stream_handler=...)` also runs workflow-side against replayed model events.

Because the model stream is consumed inside the step, cancelling it from the workflow side (e.g. with [ AgentStream.cancel()](/docs/ai/api/pydantic-ai/result/#pydantic_ai.result.AgentStream.cancel)) is not available across the durable boundary.

`Agent.run_stream_sync()` is not for workflow code: it requires no running event loop and wraps `run_stream()`. Under [ DBOSDurability](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.dbos.DBOSDurability), use the buffered async streaming APIs above or 

`Agent.run()` with an event stream handler. Outside a workflow, an agent with `DBOSDurability` behaves like a normal agent, so `run_stream_sync()` works as usual. (Wrapper `DBOSAgent` forbids `run_stream` inside workflows — use `run` + event stream handler there.)When a provider pauses a model turn mid-flight (Anthropic `pause_turn`) or runs it as a server-side job that’s polled until it’s ready ([OpenAI background mode](/docs/ai/models/openai#background-mode)), each segment runs in a separate model request step. The suspended [ ModelResponse](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse) and background job ID are checkpointed between segments, while the final response is merged and usage is recorded once. A 

[ending in a suspended response is passed to the first step. Size step timeouts for one provider round trip. If an error abandons a suspended job, its provider teardown runs in a dedicated cancellation step.](/docs/ai/core-concepts/message-history)

`message_history`Under DBOS, tools are executed in parallel by default to minimize latency. To guarantee deterministic replay and reliable recovery, DBOS waits for all parallel tool calls to complete before emitting events **in order**.
It’s equivalent to the behavior of [ with agent.parallel_tool_call_execution_mode('parallel_ordered_events')](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.parallel_tool_call_execution_mode).

If you prefer strict ordering, you can configure the agent to run tools sequentially by setting `parallel_execution_mode='sequential'` on [ DBOSDurability](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.dbos.DBOSDurability).

Additional toolsets can be passed per run via `agent.run(toolsets=...)`. Non-executing toolsets like [ ExternalToolset](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.ExternalToolset), and 

[s whose tools DBOS runs inline, are supported.](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset)

`FunctionToolset`[s and dynamic toolsets must be set when constructing the agent so their steps are registered before the workflow runs; passing them at runtime raises a](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset)

`MCPToolset``UserError`.You can customize DBOS step behavior, such as retries, by passing [ StepConfig](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.dbos.StepConfig) objects to the 

[constructor:](/docs/ai/api/pydantic-ai/durable_exec/#pydantic_ai.durable_exec.dbos.DBOSDurability)

`DBOSDurability`- `mcp_step_config`: The DBOS step config to use for MCP server communication. No retries if omitted.
- `model_step_config`: The DBOS step config to use for model request steps. No retries if omitted.
- `event_stream_handler_step_config`: The DBOS step config to use for event stream handler steps (- `DBOSDurability`only). No retries if omitted.

For custom tools, you can annotate them directly with [ @DBOS.step](https://docs.dbos.dev/python/reference/decorators#step) or 

[decorators as needed. These decorators have no effect outside DBOS workflows, so tools remain usable in non-DBOS agents.](https://docs.dbos.dev/python/reference/decorators#workflow)

`@DBOS.workflow`On top of the automatic retries for request failures that DBOS will perform, Pydantic AI and various provider API clients also have their own request retry logic. Enabling these at the same time may cause the request to be retried more often than expected, with improper `Retry-After` handling.

When using DBOS, it’s recommended to not use [HTTP Request Retries](/docs/ai/models/http-request-retries) and to turn off your provider API client’s own retry logic, for example by setting `max_retries=0` on a [custom  OpenAIProvider API client](/docs/ai/models/openai#custom-openai-client).

You can customize DBOS’s retry policy using [step configuration](#step-configuration).

DBOS can be configured to generate OpenTelemetry spans for each workflow and step execution, and Pydantic AI emits spans for each agent run, model request, and tool invocation. You can send these spans to [Pydantic Logfire](/docs/ai/integrations/logfire) to get a full, end-to-end view of what’s happening in your application.

For more information about DBOS logging and tracing, please see the [DBOS docs](https://docs.dbos.dev/python/tutorials/logging-and-tracing) for details.

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/durable_execution/dbos
