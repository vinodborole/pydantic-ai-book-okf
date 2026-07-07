---
type: Web Page
title: Prefect | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/durable_execution/prefect
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Prefect

Prefect is a workflow orchestration framework for building resilient data pipelines in Python, natively integrated with Pydantic AI.

Prefect 3.0 brings transactional semantics to your Python workflows, allowing you to group tasks into atomic units and define failure modes. If any part of a transaction fails, the entire transaction can be rolled back to a clean state.

- **Flows**are the top-level entry points for your workflow. They can contain tasks and other flows.
- **Tasks**are individual units of work that can be retried, cached, and monitored independently.

Prefect 3.0’s approach to transactional orchestration makes your workflows automatically **idempotent**: rerunnable without duplication or inconsistency across any environment. Every task is executed within a transaction that governs when and where the task’s result record is persisted. If the task runs again under an identical context, it will not re-execute but instead load its previous result.

The diagram below shows the overall architecture of an agentic application with Prefect. Prefect uses client-side task orchestration by default, with optional server connectivity for advanced features like scheduling and monitoring.

```
            +---------------------+
            |   Prefect Server    |      (Monitoring,
            |      or Cloud       |       scheduling, UI,
            +---------------------+       orchestration)
                     ^
                     |
        Flow state,  |   Schedule flows,
        metadata,    |   track execution
        logs         |
                     |
+------------------------------------------------------+
|               Application Process                    |
|   +----------------------------------------------+   |
|   |              Flow (Agent.run)                |   |
|   +----------------------------------------------+   |
|          |          |                |               |
|          v          v                v               |
|   +-----------+ +------------+ +-------------+       |
|   |   Task    | |    Task    | |    Task     |       |
|   |  (Tool)   | | (MCP Tool) | | (Model API) |       |
|   +-----------+ +------------+ +-------------+       |
|         |           |                |               |
|       Cache &     Cache &          Cache &           |
|       persist     persist          persist           |
|         to           to               to             |
|         v            v                v              |
|   +----------------------------------------------+   |
|   |     Result Storage (Local FS, S3, etc.)     |    |
|   +----------------------------------------------+   |
+------------------------------------------------------+
          |           |                |
          v           v                v
      [External APIs, services, databases, etc.]
```
See the Prefect documentation for more information.

Any agent can be wrapped in a `PrefectAgent` to get durable execution. `PrefectAgent` automatically:

- Wraps `Agent.run`and`Agent.run_sync`as Prefect flows.
- Wraps model requests as Prefect tasks.
- Wraps tool calls as Prefect tasks (configurable per-tool).
- Wraps MCP communication as Prefect tasks.

Event stream handlers are **automatically wrapped** by Prefect when running inside a Prefect flow. Each event from the stream is processed in a separate Prefect task for durability. You can customize the task behavior using the `event_stream_handler_task_config` parameter when creating the `PrefectAgent`. Do **not** manually decorate event stream handlers with `@task`. For examples, see the streaming docs

The original agent, model, and MCP server can still be used as normal outside the Prefect flow.

Here is a simple but complete example of wrapping an agent for durable execution. All it requires is to install Pydantic AI with Prefect:

Or if you’re using the slim package, you can install it with the `prefect` optional group:

The agent's `name` is used to uniquely identify its flows and tasks.

Wrapping the agent with `PrefectAgent` enables durable execution for all agent runs.

`PrefectAgent.run()` works like `Agent.run()`, but runs as a Prefect flow and executes model requests, decorated tool calls, and MCP communication as Prefect tasks.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

For more information on how to use Prefect in Python applications, see their Python documentation.

When using Prefect with Pydantic AI agents, there are a few important considerations to ensure workflows behave correctly.

Each agent instance must have a unique `name` so Prefect can correctly identify and track its flows and tasks.

Agent tools are automatically wrapped as Prefect tasks, which means they benefit from:

- **Retry logic**: Failed tool calls can be retried automatically
- **Caching**: Tool results are cached based on their inputs
- **Observability**: Tool execution is tracked in the Prefect UI

You can customize tool task behavior using `tool_task_config` (applies to all tools) or `tool_task_config_by_name` (per-tool configuration):

Set a tool’s config to `None` in `tool_task_config_by_name` to disable task wrapping for that specific tool.

When running inside a Prefect flow, `Agent.run_stream()` works but doesn’t provide real-time streaming because Prefect tasks consume their entire execution before returning results. The method will execute fully and return the complete result at once.

For real-time streaming behavior inside Prefect flows, you can set an `event_stream_handler` on the `Agent` or `PrefectAgent` instance and use `PrefectAgent.run()`.

**Note**: Event stream handlers behave differently when running inside a Prefect flow versus outside:

- **Outside a flow**: The handler receives events as they stream from the model
- **Inside a flow**: Each event is wrapped as a Prefect task for durability, which may affect timing but ensures reliability

The event stream handler function will receive the agent run context and an async iterable of events from the model’s streaming response and the agent’s execution of tools. For examples, see the streaming docs.

Additional toolsets can be passed per run via `PrefectAgent.run(toolsets=...)`, but only non-executing toolsets like `ExternalToolset`, whose tools are executed outside the agent run, are supported. Executing toolsets (`FunctionToolset` and `MCPToolset`) and dynamic toolsets must be set when constructing the agent so their tasks are registered before the flow runs; passing them at runtime raises a `UserError`.

You can customize Prefect task behavior, such as retries and timeouts, by passing `TaskConfig` objects to the `PrefectAgent` constructor:

- `mcp_task_config`: Configuration for MCP server communication tasks
- `model_task_config`: Configuration for model request tasks
- `tool_task_config`: Default configuration for all tool calls
- `tool_task_config_by_name`: Per-tool task configuration (overrides- `tool_task_config`)
- `event_stream_handler_task_config`: Configuration for event stream handler tasks (applies when running inside a Prefect flow)

Available `TaskConfig` options:

- `retries`: Maximum number of retries for the task (default:- `0`)
- `retry_delay_seconds`: Delay between retries in seconds (can be a single value or list for exponential backoff, default:- `1.0`)
- `timeout_seconds`: Maximum time in seconds for the task to complete
- `cache_policy`: Custom Prefect cache policy for the task
- `persist_result`: Whether to persist the task result
- `result_storage`: Prefect result storage for the task (e.g.,- `'s3-bucket/my-storage'`or a- `WritableFileSystem`block)
- `log_prints`: Whether to log print statements from the task (default:- `False`)

Example:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Pydantic AI and provider API clients have their own retry logic. When using Prefect, you may want to:

- Disable HTTP Request Retries in Pydantic AI
- Turn off your provider API client’s retry logic (e.g., `max_retries=0`on a custom OpenAI client)
- Rely on Prefect’s task-level retry configuration for consistency

This prevents requests from being retried multiple times at different layers.

Prefect 3.0 provides built-in caching and transactional semantics. Tasks with identical inputs will not re-execute if their results are already cached, making workflows naturally idempotent and resilient to failures.

- **Task inputs**: Messages, settings, parameters, tool arguments, and serializable dependencies

**Note**: For user dependencies to be included in cache keys, they must be serializable (e.g., Pydantic models or basic Python types). Non-serializable dependencies are automatically excluded from cache computation.

Prefect provides a built-in UI for monitoring flow runs, task executions, and failures. You can:

- View real-time flow run status
- Debug failures with full stack traces
- Set up alerts and notifications

To access the Prefect UI, you can either:

- Use Prefect Cloud (managed service)
- Run a local Prefect server with `prefect server start`

You can also use Pydantic Logfire for detailed observability. When using both Prefect and Logfire, you’ll get complementary views:

- **Prefect**: Workflow-level orchestration, task status, and retry history
- **Logfire**: Fine-grained tracing of agent runs, model requests, and tool invocations

When using Logfire with Prefect, you can enable distributed tracing to see spans for your Prefect runs included with your agent runs, model requests, and tool invocations.

For more information about Prefect monitoring, see the Prefect documentation.

To deploy and schedule a `PrefectAgent`, wrap it in a Prefect flow and use the flow’s `serve()` or `deploy()` methods:

Each flow run executes in an isolated process, and all inputs and dependencies must be serializable. Because Agent instances cannot be serialized, instantiate the agent inside the flow rather than at the module level.

The `serve()` method accepts scheduling options:

- `cron`- `'0 9 * * *'`for daily at 9am)
- `interval`
- `rrule`

For production deployments with Docker, Kubernetes, or other infrastructure, use the flow’s `deploy()` method. See the Prefect deployment documentation for more information.

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/durable_execution/prefect
