---
type: Web Page
title: Temporal | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/durable_execution/temporal
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Temporal

Temporal is a popular durable execution platform that’s natively supported by Pydantic AI.

In Temporal’s durable execution implementation, a program that crashes or encounters an exception while interacting with a model or API will retry until it can successfully complete.

Temporal relies primarily on a replay mechanism to recover from failures. As the program makes progress, Temporal saves key inputs and decisions, allowing a re-started program to pick up right where it left off.

The key to making this work is to separate the application’s repeatable (deterministic) and non-repeatable (non-deterministic) parts:

- Deterministic pieces, termed **workflows**, execute the same way when re-run with the same inputs.
- Non-deterministic pieces, termed **activities**, can run arbitrary code, performing I/O and any other operations.

Workflow code can run for extended periods and, if interrupted, resume exactly where it left off.
Critically, workflow code generally *cannot* include any kind of I/O, over the network, disk, etc.
Activity code faces no restrictions on I/O or external interactions, but if an activity fails part-way through it is restarted from the beginning.

In the case of Pydantic AI agents, integration with Temporal means that model requests, tool calls that may require I/O, and MCP server communication all need to be offloaded to Temporal activities due to their I/O requirements, while the logic that coordinates them (i.e. the agent run) lives in the workflow. Code that handles a scheduled job or web request can then execute the workflow, which will in turn execute the activities as needed.

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
See the Temporal documentation for more information.

Any agent can be wrapped in a `TemporalAgent` to get a durable agent that can be used inside a deterministic Temporal workflow, by automatically offloading all work that requires I/O (namely model requests, tool calls, and MCP server communication) to non-deterministic activities.

At the time of wrapping, the agent’s model and toolsets (including function tools registered on the agent and MCP servers) are frozen, activities are dynamically created for each, and the original model and toolsets are wrapped to call on the worker to execute the corresponding activities instead of directly performing the actions inside the workflow. The original agent can still be used as normal outside the Temporal workflow, but any changes to its model or toolsets after wrapping will not be reflected in the durable agent.

Here is a simple but complete example of wrapping an agent for durable execution, creating a Temporal workflow with durable execution logic, connecting to a Temporal server, and running the workflow from non-durable code. All it requires is a Temporal server to be running locally:

The original `Agent` cannot be used inside a deterministic Temporal workflow, but the `TemporalAgent` can.

As explained above, the workflow represents a deterministic piece of code that can use non-deterministic activities for operations that require I/O. Subclassing `PydanticAIWorkflow` is optional but provides proper typing for the `__pydantic_ai_agents__` class variable.

List the `TemporalAgent`s used by this workflow. The `PydanticAIPlugin` will automatically register their activities with the worker. Alternatively, if modifying the worker initialization is easier than the workflow class, you can use `AgentPlugin` to register agents directly on the worker.

`TemporalAgent.run()` works just like `Agent.run()`, but it will automatically offload model requests, tool calls, and MCP server communication to Temporal activities.

We connect to the Temporal server which keeps track of workflow and activity execution.

This assumes the Temporal server is running locally.

The `PydanticAIPlugin` tells Temporal to use Pydantic for serialization and deserialization, treats `UserError` exceptions as non-retryable, and automatically registers activities for agents listed in `__pydantic_ai_agents__`.

We start the worker that will listen on the specified task queue and run workflows and activities. In a real world application, this might be run in a separate service.

The agent's `name` is used to uniquely identify its activities.

We call on the server to execute the workflow on a worker that's listening on the specified task queue.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

In a real world application, the agent, workflow, and worker are typically defined separately from the code that calls for a workflow to be executed.
Because Temporal workflows need to be defined at the top level of the file and the `TemporalAgent` instance is needed inside the workflow and when starting the worker (to register the activities), it needs to be defined at the top level of the file as well.

For more information on how to use Temporal in Python applications, see their Python SDK guide.

There are a few considerations specific to agents and toolsets when using Temporal for durable execution. These are important to understand to ensure that your agents and toolsets work correctly with Temporal’s workflow and activity model.

To ensure that Temporal knows what code to run when an activity fails or is interrupted and then restarted, even if your code is changed in between, each activity needs to have a name that’s stable and unique.

When `TemporalAgent` dynamically creates activities for the wrapped agent’s model requests and toolsets (specifically those that implement their own tool listing and calling, i.e. `FunctionToolset` and `MCPToolset`), their names are derived from the agent’s `name` and the toolsets’ `id`s. These fields are normally optional, but are required to be set when using Temporal. They should not be changed once the durable agent has been deployed to production as this would break active workflows.

For dynamic toolsets created with the `@agent.toolset` decorator, the `id` parameter must be set explicitly. Note that with Temporal, `per_run_step=False` is not respected, as the toolset always needs to be created on-the-fly in the activity.

Other than that, any agent and toolset will just work!

As workflows and activities run in separate processes, any values passed between them need to be serializable. As these payloads are stored in the workflow execution event history, Temporal limits their size to 2MB.

To account for these limitations, tool functions and the event stream handler running inside activities receive a limited version of the agent’s `RunContext`, and it’s your responsibility to make sure that the dependencies object provided to `TemporalAgent.run()` can be serialized using Pydantic.

Specifically, only the `deps`, `run_id`, `metadata`, `retries`, `tool_call_id`, `tool_name`, `tool_call_approved`, `tool_call_metadata`, `retry`, `max_retries`, `run_step`, `usage`, and `partial_output` fields are available by default, and trying to access `model`, `prompt`, `messages`, or `tracer` will raise an error.
If you need one or more of these attributes to be available inside activities, you can create a `TemporalRunContext` subclass with custom `serialize_run_context` and `deserialize_run_context` class methods and pass it to `TemporalAgent` as `run_context_type`.

Because Temporal activities cannot stream output directly to the activity call site, `Agent.run_stream()`, `Agent.run_stream_events()`, and `Agent.iter()` are not supported.

Instead, you can implement streaming by setting an `event_stream_handler` on the `Agent` or `TemporalAgent` instance and using `TemporalAgent.run()` inside the workflow.
The event stream handler function will receive the agent run context and an async iterable of events from the model’s streaming response and the agent’s execution of tools. For examples, see the streaming docs.

As the streaming model request activity, workflow, and workflow execution call all take place in separate processes, passing data between them requires some care:

- To get data from the workflow call site or workflow to the event stream handler, you can use a dependencies object.
- To get data from the event stream handler to the workflow, workflow call site, or a frontend, you need to use an external system that the event stream handler can write to and the event consumer can read from, like a message queue. You can use the dependency object to make sure the same connection string or other unique ID is available in all the places that need it.

`Agent.run(model=...)` normally supports both model strings (like `'openai:gpt-5.2'`) and model instances. However, `TemporalAgent` does not support arbitrary model instances because they cannot be serialized for Temporal’s replay mechanism.

To use model instances with `TemporalAgent`, you need to pre-register them by passing a dict of model instances to `TemporalAgent(models={...})`. You can then reference them by name or by passing the registered instance directly. If the wrapped agent doesn’t have a model set, the first registered model will be used as the default.

Model strings work as expected. For scenarios where you need to customize the provider used by the model string (e.g., inject API keys from deps), you can pass a `provider_factory` to `TemporalAgent`, which is passed the `RunContext` and provider name.

Here’s an example showing how to pre-register and use multiple models:

Additional toolsets can be passed per run via `TemporalAgent.run(toolsets=...)`, but only non-executing toolsets like `ExternalToolset`, whose tools are executed outside the agent run, are supported. Executing toolsets (`FunctionToolset` and `MCPToolset`) and dynamic toolsets must be set when constructing the agent so their activities can be registered with the worker before the workflow runs; passing them at runtime raises a `UserError`.

Temporal activity configuration, like timeouts and retry policies, can be customized by passing `temporalio.workflow.ActivityConfig` objects to the `TemporalAgent` constructor:

- 
`activity_config`: The base Temporal activity config to use for all activities. If no config is provided, a`start_to_close_timeout`of 60 seconds is used.
- 
`model_activity_config`: The Temporal activity config to use for model request activities. This is merged with the base activity config.
- 
`toolset_activity_config`: The Temporal activity config to use for get-tools and call-tool activities for specific toolsets identified by ID. This is merged with the base activity config.
- 
`tool_activity_config`: The Temporal activity config to use for specific tool call activities identified by toolset ID and tool name. This is merged with the base and toolset-specific activity configs.If a tool does not use I/O, you can specify `False`to disable using an activity. Note that the tool is required to be defined as an`async`function as non-async tools are run in threads which are non-deterministic and thus not supported outside of activities.

On top of the automatic retries for request failures that Temporal will perform, Pydantic AI and various provider API clients also have their own request retry logic. Enabling these at the same time may cause the request to be retried more often than expected, with improper `Retry-After` handling.

When using Temporal, it’s recommended to not use HTTP Request Retries and to turn off your provider API client’s own retry logic, for example by setting `max_retries=0` on a custom `OpenAIProvider` API client.

You can customize Temporal’s retry policy using activity configuration.

Temporal generates telemetry events and metrics for each workflow and activity execution, and Pydantic AI generates events for each agent run, model request and tool call. These can be sent to Pydantic Logfire to get a complete picture of what’s happening in your application.

To use Logfire with Temporal, you need to pass a `LogfirePlugin` object to Temporal’s `Client.connect()`:

By default, the `LogfirePlugin` will instrument Temporal (including metrics) and Pydantic AI and send all data to Logfire. To customize Logfire configuration and instrumentation, you can pass a `logfire_setup` function to the `LogfirePlugin` constructor and return a custom `Logfire` instance (i.e. the result of `logfire.configure()`). To disable sending Temporal metrics to Logfire, you can pass `metrics=False` to the `LogfirePlugin` constructor.

When `logfire.info` is used inside an activity and the `pandas` package is among your project’s dependencies, you may encounter the following error which seems to be the result of an import race condition:

```
AttributeError: partially initialized module 'pandas' has no attribute '_pandas_parser_CAPI' (most likely due to a circular import)
```
To fix this, you can use the `temporalio.workflow.unsafe.imports_passed_through()` context manager to proactively import the package and not have it be reloaded in the workflow sandbox:

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/durable_execution/temporal
