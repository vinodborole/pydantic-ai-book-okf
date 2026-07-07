---
type: Web Page
title: DBOS | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/durable_execution/dbos
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# DBOS

DBOS is a lightweight durable execution library natively integrated with Pydantic AI.

DBOS workflows make your program **durable** by checkpointing its state in a database. If your program ever fails, when it restarts all your workflows will automatically resume from the last completed step.

- **Workflows**must be deterministic and generally cannot include I/O.
- **Steps**may perform I/O (network, disk, API calls). If a step fails, it restarts from the beginning.

Every workflow input and step output is durably stored in the system database. When workflow execution fails, whether from crashes, network issues, or server restarts, DBOS leverages these checkpoints to recover workflows from their last completed step.

DBOS **queues** provide durable, database-backed alternatives to systems like Celery or BullMQ, supporting features such as concurrency limits, rate limits, timeouts, and prioritization. See the DBOS docs for details.

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
See the DBOS documentation for more information.

Any agent can be wrapped in a `DBOSAgent` to get durable execution. `DBOSAgent` automatically:,

- Wraps `Agent.run`and`Agent.run_sync`as DBOS workflows.
- Wraps model requests and MCP communication as DBOS steps.

Custom tool functions and event stream handlers are **not automatically wrapped** by DBOS.
If they involve non-deterministic behavior or perform I/O, you should explicitly decorate them with `@DBOS.step`.

The original agent, model, and MCP server can still be used as normal outside the DBOS workflow.

Here is a simple but complete example of wrapping an agent for durable execution. All it requires is to install Pydantic AI with the DBOS open-source library:

Or if you’re using the slim package, you can install it with the `dbos` optional group:

Workflows and `DBOSAgent` must be defined before `DBOS.launch()` so that recovery can correctly find all workflows.

`DBOSAgent.run()` works like `Agent.run()`, but runs as a DBOS workflow and executes model requests, decorated tool calls, and MCP communication as DBOS steps.

This example uses SQLite. Postgres is recommended for production.

The agent's `name` is used to uniquely identify its workflows.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Because DBOS workflows need to be defined before calling `DBOS.launch()` and the `DBOSAgent` instance automatically registers `run` and `run_sync` as workflows, it needs to be defined before calling `DBOS.launch()` as well.

For more information on how to use DBOS in Python applications, see their Python SDK guide.

When using DBOS with Pydantic AI agents, there are a few important considerations to ensure workflows and toolsets behave correctly.

Each agent instance must have a unique `name` so DBOS can correctly resume workflows after a failure or restart.

Tools and event stream handlers are not automatically wrapped by DBOS. You can decide how to integrate them:

- Decorate with `@DBOS.step`if the function involves non-determinism or I/O.
- Skip the decorator if durability isn’t needed, so you avoid the extra DB checkpoint write.
- If the function needs to enqueue tasks or invoke other DBOS workflows, run it inside the agent’s main workflow (not as a step).

Other than that, any agent and toolset will just work!

DBOS checkpoints workflow inputs/outputs and step outputs into a database using `pickle`. This means you need to make sure dependencies object provided to `DBOSAgent.run()` or `DBOSAgent.run_sync()`, and tool outputs can be serialized using pickle. You may also want to keep the inputs and outputs small (under ~2 MB). PostgreSQL and SQLite support up to 1 GB per field, but large objects may impact performance.

Because DBOS cannot stream output directly to the workflow or step call site, `Agent.run_stream()` and `Agent.run_stream_events()` are not supported when running inside of a DBOS workflow.

Instead, you can implement streaming by setting an `event_stream_handler` on the `Agent` or `DBOSAgent` instance and using `DBOSAgent.run()`.
The event stream handler function will receive the agent run context and an async iterable of events from the model’s streaming response and the agent’s execution of tools. For examples, see the streaming docs.

When using `DBOSAgent`, tools are executed in parallel by default to minimize latency. To guarantee deterministic replay and reliable recovery, DBOS waits for all parallel tool calls to complete before emitting events **in order**.
It’s equivalent to the behavior of `with agent.parallel_tool_call_execution_mode('parallel_ordered_events')`.

If you prefer strict ordering, you can configure the agent to run tools sequentially by setting `parallel_execution_mode='sequential'` when initializing the `DBOSAgent`.

Additional toolsets can be passed per run via `DBOSAgent.run(toolsets=...)`. Non-executing toolsets like `ExternalToolset`, and `FunctionToolset`s whose tools DBOS runs inline, are supported. `MCPToolset`s and dynamic toolsets must be set when constructing the agent so their steps are registered before the workflow runs; passing them at runtime raises a `UserError`.

You can customize DBOS step behavior, such as retries, by passing `StepConfig` objects to the `DBOSAgent` constructor:

- `mcp_step_config`: The DBOS step config to use for MCP server communication. No retries if omitted.
- `model_step_config`: The DBOS step config to use for model request steps. No retries if omitted.

For custom tools, you can annotate them directly with `@DBOS.step` or `@DBOS.workflow` decorators as needed. These decorators have no effect outside DBOS workflows, so tools remain usable in non-DBOS agents.

On top of the automatic retries for request failures that DBOS will perform, Pydantic AI and various provider API clients also have their own request retry logic. Enabling these at the same time may cause the request to be retried more often than expected, with improper `Retry-After` handling.

When using DBOS, it’s recommended to not use HTTP Request Retries and to turn off your provider API client’s own retry logic, for example by setting `max_retries=0` on a custom `OpenAIProvider` API client.

You can customize DBOS’s retry policy using step configuration.

DBOS can be configured to generate OpenTelemetry spans for each workflow and step execution, and Pydantic AI emits spans for each agent run, model request, and tool invocation. You can send these spans to Pydantic Logfire to get a full, end-to-end view of what’s happening in your application.

For more information about DBOS logging and tracing, please see the DBOS docs for details.

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/durable_execution/dbos
