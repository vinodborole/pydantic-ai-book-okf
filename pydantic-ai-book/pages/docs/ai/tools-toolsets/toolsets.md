---
type: Web Page
title: Toolsets | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/toolsets
timestamp: '2026-08-10T07:48:56.025339+00:00'
---

# Toolsets

A toolset represents a collection of [tools](/docs/ai/tools-toolsets/tools) that can be registered with an agent in one go. They can be reused by different agents, swapped out at runtime or during testing, and composed in order to dynamically filter which tools are available, modify tool definitions, or change tool execution behavior. A toolset can contain locally defined functions, depend on an external service to provide them, or implement custom logic to list available tools and handle them being called. Toolsets can also be provided via [capabilities](/docs/ai/capabilities/overview), which bundle tools with hooks, instructions, and model settings.

Toolsets are used (among many other things) to define [MCP servers](/docs/ai/mcp/client) available to an agent. Pydantic AI includes many kinds of toolsets which are described below, and you can define a [custom toolset](#building-a-custom-toolset) by inheriting from the [`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset) class.

The toolsets that will be available during an agent run can be specified in four different ways:

- at agent construction time, via the [`toolsets`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) keyword argument to`Agent` , which takes toolset instances as well as functions that generate toolsets[dynamically](#dynamically-building-a-toolset) based on the agent[run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)
- at agent run time, via the `toolsets` keyword argument to[`agent.run()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run) ,[`agent.run_sync()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_sync) ,[`agent.run_stream()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run_stream) , or[`agent.iter()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.iter) . These toolsets will be additional to those registered on the`Agent`
- [dynamically](#dynamically-building-a-toolset) , via the[`@agent.toolset`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.toolset) decorator which lets you build a toolset based on the agent[run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)
- as a contextual override, via the `toolsets` keyword argument to the[`agent.override()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.override) context manager. These toolsets will replace those provided at agent construction or run time during the life of the context manager

The [`FunctionToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset) will be explained in detail in the next section.

We're using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here because it makes it easy to see which tools were available on each run.

This `extra_toolset` will be ignored because we're inside an override context.

*(This example is complete, it can be run “as is”)*

As the name suggests, a [`FunctionToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset) makes locally defined functions available as tools.

Functions can be added as tools in four different ways:

- via the [`@toolset.tool`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.tool) decorator — for tools that need access to the agent[context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)
- via the [`@toolset.tool_plain`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.tool_plain) decorator — for tools that do not need access to the agent[context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)
- via the [`tools`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.__init__) keyword argument to the constructor which can take either plain functions, or instances of[`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool)
- via the [`toolset.add_function()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.add_function) and[`toolset.add_tool()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.add_tool) methods which can take a plain function or an instance of[`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) respectively

The `add_function()` and `add_tool()` methods can also be used from a tool function to dynamically register new tools during a run to be available in future run steps.

We're using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here because it makes it easy to see which tools were available on each run.

*(This example is complete, it can be run “as is”)*

A [`FunctionToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset) can provide instructions that are automatically included in the model request. This lets each toolset carry its own usage guidance alongside its tools, so you don’t need to duplicate instructions on every agent that uses the toolset.

Instructions can be provided as strings, functions (sync or async, with or without [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)), or a mix of both:

*(This example is complete, it can be run “as is”)*

You can also use the [`@toolset.instructions`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.instructions) decorator to register dynamic instruction functions that can access the run context:

*(This example is complete, it can be run “as is”)*

When a toolset with instructions is used alongside agent-level [`instructions`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__), the toolset instructions are appended after the agent instructions:

*(This example is complete, it can be run “as is”)*

When multiple toolsets with instructions are registered on an agent, all their instructions are combined:

*(This example is complete, it can be run “as is”)*

Toolsets can be composed to dynamically filter which tools are available, modify tool definitions, or change tool execution behavior. Multiple toolsets can also be combined into one.

[`CombinedToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.CombinedToolset) takes a list of toolsets and lets them be used as one.

We're using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here because it makes it easy to see which tools were available on each run.

*(This example is complete, it can be run “as is”)*

[`FilteredToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FilteredToolset) wraps a toolset and filters available tools ahead of each step of the run based on a user-defined function that is passed the agent [run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and each tool’s [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition) and returns a boolean to indicate whether or not a given tool should be available.

To easily chain different modifications, you can also call [`filtered()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.filtered) on any toolset instead of directly constructing a `FilteredToolset`.

We're using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here because it makes it easy to see which tools were available on each run.

*(This example is complete, it can be run “as is”)*

[`PrefixedToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.PrefixedToolset) wraps a toolset and adds a prefix to each tool name to prevent tool name conflicts between different toolsets.

To easily chain different modifications, you can also call [`prefixed()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.prefixed) on any toolset instead of directly constructing a `PrefixedToolset`.

We're using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here because it makes it easy to see which tools were available on each run.

*(This example is complete, it can be run “as is”)*

[`RenamedToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.RenamedToolset) wraps a toolset and lets you rename tools using a dictionary mapping new names to original names. This is useful when the names provided by a toolset are ambiguous or would conflict with tools defined by other toolsets, but [prefixing them](#prefixing-tool-names) creates a name that is unnecessarily long or could be confusing to the model.

To easily chain different modifications, you can also call [`renamed()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.renamed) on any toolset instead of directly constructing a `RenamedToolset`.

We're using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here because it makes it easy to see which tools were available on each run.

*(This example is complete, it can be run “as is”)*

[`PreparedToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.PreparedToolset) lets you modify the entire list of available tools ahead of each step of the agent run using a user-defined function that takes the agent [run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and a list of [`ToolDefinition`s](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition) and returns the tool definitions to expose for that step.

This is the toolset-specific equivalent of the [`prepare_tools`](/docs/ai/tools-toolsets/tools-advanced#prepare-tools) capability hook that prepares all tool definitions registered on an agent across toolsets.

Note that it is not possible to add or rename tools using `PreparedToolset`. Instead, you can use [`FunctionToolset.add_function()`](#function-toolset) or [`RenamedToolset`](#renaming-tools).

To easily chain different modifications, you can also call [`prepared()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.prepared) on any toolset instead of directly constructing a `PreparedToolset`.

We're using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here because it makes it easy to see which tools were available on each run.

[`ApprovalRequiredToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.ApprovalRequiredToolset) wraps a toolset and lets you dynamically [require approval](/docs/ai/tools-toolsets/deferred-tools#human-in-the-loop-tool-approval) for a given tool call based on a user-defined function that is passed the agent [run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext), the tool’s [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition), and the validated tool call arguments. If no function is provided, all tool calls will require approval.

To easily chain different modifications, you can also call [`approval_required()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.approval_required) on any toolset instead of directly constructing a `ApprovalRequiredToolset`.

See the [Human-in-the-Loop Tool Approval](/docs/ai/tools-toolsets/deferred-tools#human-in-the-loop-tool-approval) documentation for more information on how to handle agent runs that call tools that require approval and how to pass in the results.

We're using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here because it makes it easy to specify which tools to call.

*(This example is complete, it can be run “as is”)*

[`DeferredLoadingToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.DeferredLoadingToolset) wraps a toolset and marks its tools for deferred loading, hiding them from the model until discovered via [tool search](/docs/ai/tools-toolsets/tools-advanced#tool-search). This is useful for large toolsets (e.g. MCP servers with many endpoints) where loading all tool definitions into the model’s context would be wasteful.

[`FunctionToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset) also accepts `defer_loading=True` in its constructor to mark all tools for deferred loading. For other toolsets, call [`.defer_loading()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.defer_loading) — pass a list of tool names to hide only specific tools, or `None` (the default) to hide all.

[`IncludeReturnSchemasToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.IncludeReturnSchemasToolset) wraps a toolset and sets `include_return_schema=True` on all its tools, causing the model to receive return type information. For models that natively support return schemas (e.g. Google Gemini), the schema is passed as a structured API field. For other models, it is injected into the tool description as JSON text.

To easily chain different modifications, you can also call [`.include_return_schemas()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.include_return_schemas) on any toolset instead of directly constructing an `IncludeReturnSchemasToolset`.

*(This example is complete, it can be run “as is”)*

This is the toolset-level equivalent of the [`IncludeToolReturnSchemas`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.IncludeToolReturnSchemas) capability, which applies across all toolsets or a selected subset.

[`SetMetadataToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.SetMetadataToolset) wraps a toolset and merges metadata key-value pairs onto all its tools. This is useful for tagging tools with configuration that other capabilities or custom logic can inspect.

To easily chain different modifications, you can also call [`.with_metadata()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.with_metadata) on any toolset instead of directly constructing a `SetMetadataToolset`.

*(This example is complete, it can be run “as is”)*

This is the toolset-level equivalent of the [`SetToolMetadata`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.SetToolMetadata) capability, which applies across all toolsets or a selected subset.

[`WrapperToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.WrapperToolset) wraps another toolset and delegates all responsibility to it.

It is a no-op by default, but you can subclass `WrapperToolset` to change the wrapped toolset’s tool execution behavior by overriding the [`call_tool()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.call_tool) method.

All docs examples are tested in CI and their output is verified, so we need `LOG` to always have the same order whenever this code is run. Since the tools could finish in any order, we sleep an increasing amount of time based on which number tool call we are to ensure that they finish (and log) in the same order they were called in.

We use [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here as it will automatically call each tool.

*(This example is complete, it can be run “as is”)*

If your agent needs to be able to call [external tools](/docs/ai/tools-toolsets/deferred-tools#external-tool-execution) that are provided and executed by an upstream service or frontend, you can build an [`ExternalToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.ExternalToolset) from a list of [`ToolDefinition`s](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition) containing the tool names, arguments JSON schemas, and descriptions.

When the model calls an external tool, the call is considered to be [“deferred”](/docs/ai/tools-toolsets/deferred-tools#deferred-tools), and the agent run will end with a [`DeferredToolRequests`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolRequests) output object with a `calls` list holding [`ToolCallPart`s](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolCallPart) containing the tool name, validated arguments, and a unique tool call ID, which are expected to be passed to the upstream service or frontend that will produce the results.

When the tool call results are received from the upstream service or frontend, you can build a [`DeferredToolResults`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.DeferredToolResults) object with a `calls` dictionary that maps each tool call ID to an arbitrary value to be returned to the model, a [`ToolReturn`](/docs/ai/tools-toolsets/tools-advanced#advanced-tool-returns) object, or an exception in case the tool call failed: a [`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) if the model should [try again](/docs/ai/tools-toolsets/tools-advanced#tool-retries), or a [`ToolFailed`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ToolFailed) if the failure should be reported to the model as a [failed result](/docs/ai/tools-toolsets/tools-advanced#tool-failed) (without consuming the tool’s retry budget) so it can decide how to proceed. This `DeferredToolResults` object can then be provided to one of the agent run methods as `deferred_tool_results`, alongside the original run’s [message history](/docs/ai/core-concepts/message-history).

Note that you need to add `DeferredToolRequests` to the `Agent`’s or `agent.run()`’s [`output_type`](/docs/ai/core-concepts/output#structured-output) so that the possible types of the agent run output are correctly inferred. For more information, see the [Deferred Tools](/docs/ai/tools-toolsets/deferred-tools#deferred-tools) documentation.

To demonstrate, let us first define a simple agent *without* deferred tools:

Next, let’s define a function that represents a hypothetical “run agent” API endpoint that can be called by the frontend and takes a list of messages to send to the model, a list of frontend tool definitions, and optional deferred tool results. This is where `ExternalToolset`, `DeferredToolRequests`, and `DeferredToolResults` come in:

As mentioned in the [Deferred Tools](/docs/ai/tools-toolsets/deferred-tools#deferred-tools) documentation, these `toolsets` are additional to those provided to the `Agent` constructor

As mentioned in the [Deferred Tools](/docs/ai/tools-toolsets/deferred-tools#deferred-tools) documentation, this `output_type` overrides the one provided to the `Agent` constructor, so we have to make sure to not lose it

We don't include an `user_prompt` keyword argument as we expect the frontend to provide it via `messages`

Now, imagine that the code below is implemented on the frontend, and `run_agent` stands in for an API call to the backend that runs the agent. This is where we actually execute the deferred tool calls and start a new run with the new result included:

Imagine that this returns the frontend [`navigator.language`](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/language).

*(This example is complete, it can be run “as is”)*

Toolsets can be built dynamically ahead of each agent run or run step using a function that takes the agent [run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and returns a toolset or `None`. This is useful when a toolset (like an MCP server) depends on information specific to an agent run, like its [dependencies](/docs/ai/core-concepts/dependencies) — for example to [connect to an MCP server with per-user credentials](/docs/ai/mcp/client#per-user-authentication).

To register a dynamic toolset, you can pass a function that takes [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) to the `toolsets` argument of the `Agent` constructor, or you can wrap a compliant function in the [`@agent.toolset`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.toolset) decorator.

By default, the function will be called again ahead of each agent run step. If you are using the decorator, you can optionally provide a `per_run_step=False` argument to indicate that the toolset only needs to be built once for the entire run.

We're using [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) here because it makes it easy to see which tools were available on each run.

We're using the agent's dependencies to give the `toggle` tool access to the `active` via the `RunContext` argument.

This shows the available tools *after* the `toggle` tool was executed, as the "last model request" was the one that returned the `toggle` tool result to the model.

*(This example is complete, it can be run “as is”)*

To define a fully custom toolset with its own logic to list available tools and handle them being called, you can subclass [`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset) and implement the [`get_tools()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.get_tools) and [`call_tool()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.call_tool) methods.

You can also override the [`get_instructions()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.get_instructions) method to provide a description of how to use the toolset’s tools. This will be injected into the agent’s instructions and is useful for helping the model understand how to effectively use your toolset’s tools.

The toolset lifecycle provides hooks for managing state at different scopes:

- [`for_run()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.for_run) : Called once before each agent run. Return a fresh instance for per-run state isolation (e.g. resetting counters, creating a new session). The framework enters and exits the returned instance.
- [`for_run_step()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.for_run_step) : Called at the start of each run step. Return a modified instance for per-step state transitions. If managing inner toolset transitions (e.g. swapping one toolset for another), you are responsible for the inner lifecycle (exiting the old, entering the new).
- [`__aenter__()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.__aenter__) and[`__aexit__()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.__aexit__) : Set up and tear down resources (e.g. network connections) that should live for the duration of the agent run.

Toolsets support lifecycle hooks for per-run isolation and per-step state management:

- [`for_run(ctx)`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.for_run) — called once per agent run, before`__aenter__` . Return a fresh instance to isolate state between runs. Default: returns`self` .
- [`for_run_step(ctx)`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.for_run_step) — called at the start of each run step. Manage internal transitions (e.g. refreshing tool availability) in-place. Default: returns`self` .

Third-party toolsets can also be wrapped as [capabilities](/docs/ai/capabilities/overview), which bundle tools with hooks, instructions, and model settings. See [Extensibility](/docs/ai/guides/extensibility) for the full ecosystem.

Pydantic AI provides [`MCPToolset`](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) for connecting to and calling tools on local and remote MCP servers, with the [`MCP` capability](/docs/ai/capabilities/mcp) as the recommended higher-level entry point. See the [MCP overview](/docs/ai/mcp/overview) and [MCP client](/docs/ai/mcp/client) documentation for details.

Toolsets that implement [Agent Skills](https://agentskills.io) support so agents can efficiently discover and perform specific tasks:

- [`pydantic-ai-skills`](https://github.com/DougTrajano/pydantic-ai-skills) -`SkillsToolset` implements Agent Skills support with progressive disclosure (load skills on-demand to reduce tokens). Supports filesystem and programmatic skills; compatible with[agentskills.io](https://agentskills.io) .

Toolsets for task planning and progress tracking help agents organize complex work and provide visibility into agent progress:

- [`pydantic-ai-todo`](https://github.com/vstorm-co/pydantic-ai-todo) -`TodoToolset` with`read_todos` and`write_todos` tools. Included in the third-party[`pydantic-deep`](https://github.com/vstorm-co/pydantic-deepagents)[deep agent](/docs/ai/guides/multi-agent-applications#deep-agents) framework.

Toolsets for file operations help agents read, write, and edit files:

- [`pydantic-ai-filesystem-sandbox`](https://github.com/zby/pydantic-ai-filesystem-sandbox) -`FileSystemToolset` with a sandbox and LLM-friendly errors
- [`pydantic-deep`](https://github.com/vstorm-co/pydantic-deepagents) — Deep agent framework that includes a`FilesystemToolset` with multiple backends (in-memory, real filesystem, Docker sandbox).

Toolsets for sandboxed code execution help agents run code in a sandboxed environment:

- [`mcp-run-python`](https://github.com/pydantic/mcp-run-python) - MCP server by the Pydantic team that runs Python code in a sandboxed environment. Can be used as`MCPToolset(StdioTransport(command='uv', args=['run', 'mcp-run-python', 'stdio']))` .

If you’d like to use tools or a [toolkit](https://python.langchain.com/docs/concepts/tools/#toolkits) from LangChain’s [community tool library](https://python.langchain.com/docs/integrations/tools/) with Pydantic AI, you can use the [`LangChainToolset`](/docs/ai/api/pydantic-ai/ext/#pydantic_ai.ext.langchain.LangChainToolset) which takes a list of LangChain tools. Note that Pydantic AI will not validate the arguments in this case — it’s up to the model to provide arguments matching the schema specified by the LangChain tool, and up to the LangChain tool to raise an error if the arguments are invalid.

You will need to install the `langchain-community` package and any others required by the tools in question.

```
from langchain_community.agent_toolkits import SlackToolkit
from pydantic_ai import Agent
from pydantic_ai.ext.langchain import LangChainToolset
toolkit = SlackToolkit()
toolset = LangChainToolset(toolkit.get_tools())
agent = Agent('openai:gpt-5.2', toolsets=[toolset])
# ...
```
[`pydantic-ai-ejentum`](https://pypi.org/project/pydantic-ai-ejentum/) wraps the [Ejentum Reasoning Harness](https://ejentum.com) as a `FunctionToolset` subclass. `EjentumToolset` registers four agent-callable tools (`harness_reasoning`, `harness_code`, `harness_anti_deception`, `harness_memory`). The agent calls one before generating; each call returns a structured cognitive scaffold (named failure pattern, executable procedure, suppression vectors, falsification test) that the model reads internally to shape its next response.

You will need to install the `pydantic-ai-ejentum` package and set your Ejentum API key in the `EJENTUM_API_KEY` environment variable (free and paid tiers at [https://ejentum.com/pricing](https://ejentum.com/pricing)), or pass `api_key=` to the constructor.

```
from pydantic_ai import Agent
from pydantic_ai_ejentum import EjentumToolset
toolset = EjentumToolset()
agent = Agent('openai:gpt-5.2', toolsets=[toolset])
```
The toolset emits Pydantic AI `instructions` that nudge the agent to call the matching `harness_*` tool before generating. Pass `add_instructions=False` to suppress and supply routing guidance from your own system prompt.

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/toolsets
