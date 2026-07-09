---
type: Web Page
title: Advanced Tool Features | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/tools-advanced
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Advanced Tool Features

This page covers advanced features for function tools in Pydantic AI. For basic tool usage, see the [Function Tools](/docs/ai/tools-toolsets/tools) documentation.

Tools can return anything that Pydantic can serialize to JSON, as well as audio, video, image or document content depending on the types of [multi-modal input](/docs/ai/advanced-features/input) the model supports:

*(This example is complete, it can be run “as is”)*

Some models (e.g. Gemini) natively support semi-structured return values, while some expect text (OpenAI) but seem to be just as good at extracting meaning from the data. If a Python object is returned and the model expects a string, the value will be serialized to JSON.

For scenarios where you need more control over both the tool’s return value and the content sent to the model, you can use [ ToolReturn](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolReturn). This is particularly useful when you want to:

- Separate the structured return value from additional content sent to the model
- Explicitly send content as a separate user message (rather than in the tool result)
- Include additional metadata that shouldn’t be sent to the LLM

Here’s an example of a computer automation tool that captures screenshots and provides visual feedback:

- `return_value`- [Tool Output](#function-tool-output)above).
- `content`- **separate user message**after the tool result. Use this when you explicitly want content to appear outside the tool result, or when combining structured return values with rich content.
- `metadata`

This separation allows you to provide rich context to the model while maintaining clean, structured return values for your application logic. For multimodal content that should be sent natively in the tool result (when supported by the model), return it directly from the tool function or include it in `return_value` (see [Tool Output](#function-tool-output) above).

If you have a function that lacks appropriate documentation (i.e. poorly named, no type information, poor docstring, use of *args or **kwargs and suchlike) then you can still turn it into a tool that can be effectively used by the agent with the [ Tool.from_schema](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool.from_schema) function. With this you provide the name, description, JSON schema, and whether the function takes a 

`RunContext` for the function directly:```
from pydantic_ai import Agent, Tool
from pydantic_ai.models.test import TestModel
def foobar(**kwargs) -> str:
    return kwargs['a'] + kwargs['b']
tool = Tool.from_schema(
    function=foobar,
    name='sum',
    description='Sum two numbers.',
    json_schema={
        'additionalProperties': False,
        'properties': {
            'a': {'description': 'the first number', 'type': 'integer'},
            'b': {'description': 'the second number', 'type': 'integer'},
        },
        'required': ['a', 'b'],
        'type': 'object',
    },
    takes_ctx=False,
)
test_model = TestModel()
agent = Agent(test_model, tools=[tool])
result = agent.run_sync('testing...')
print(result.output)
#> {"sum":0}
```
Please note that validation of the tool arguments will not be performed, and this will pass all arguments as keyword arguments.

Tools can optionally be defined with another function: `prepare`, which is called at each step of a run to
customize the definition of the tool passed to the model, or omit the tool completely from that step.

A `prepare` method can be registered via the `prepare` kwarg to any of the tool registration mechanisms:

- `@agent.tool`
- `@agent.tool_plain`
- `Tool`

The `prepare` method, should be of type [ ToolPrepareFunc](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolPrepareFunc), a function which takes 

[and a pre-built](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext`[, and should either return that](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)

`ToolDefinition``ToolDefinition` with or without modifying it, return a new `ToolDefinition`, or return `None` to indicate this tools should not be registered for that step.Here’s a simple `prepare` method that only includes the tool if the value of the dependency is `42`.

As with the previous example, we use [ TestModel](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) to demonstrate the behavior without calling a real model.

*(This example is complete, it can be run “as is”)*

Here’s a more complex example where we change the description of the `name` parameter to based on the value of `deps`

For the sake of variation, we create this tool using the [ Tool](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) dataclass.

*(This example is complete, it can be run “as is”)*

In addition to per-tool `prepare` methods, you can also define an agent-wide `prepare_tools` function. This function is called at each step of a run and allows you to filter or modify the list of all tool definitions available to the agent for that step. This is especially useful if you want to enable or disable multiple tools at once, or apply global logic based on the current context.

The `prepare_tools` function should be of type [ ToolsPrepareFunc](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolsPrepareFunc), which takes the 

[and a list of](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext`[, and returns the tool definitions to expose for that step. Return the](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition)

`ToolDefinition``tool_defs` argument to keep every tool as-is, or `[]` to expose no tools.To modify output tools, you can set a `prepare_output_tools` function instead.

Here’s an example that makes all tools strict if the model is an OpenAI model:

*(This example is complete, it can be run “as is”)*

Here’s another example that conditionally filters out the tools by name if the dependency (`ctx.deps`) is `True`:

*(This example is complete, it can be run “as is”)*

You can use `prepare_tools` to:

- Dynamically enable or disable tools based on the current model, dependencies, or other context
- Modify tool definitions globally (e.g., set all tools to strict mode, change descriptions, etc.)

If both per-tool `prepare` and agent-wide `prepare_tools` are used, the per-tool `prepare` is applied first to each tool, and then `prepare_tools` is called with the resulting list of tool definitions.

The `tool_choice` setting in [ ModelSettings](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings) controls which tools the model can use during a request. This is useful for disabling tools, forcing tool use, or restricting which tools are available.

Pydantic AI distinguishes between ** function tools** (tools you register via 

`@agent.tool`, [toolsets](/docs/ai/tools-toolsets/toolsets), or

[MCP](/docs/ai/mcp/client)), and

**output tools**(internal tools used for

[structured output](/docs/ai/core-concepts/output#tool-output)).

| Value | Description | 
|---|---|
| `'auto'`(default) | Model decides whether to use tools. All tools available. | 
| `'none'` | Disable function tools. Model can respond with text or use output tools. | 
| `'required'` | Force the model to use a function tool. Excludes output tools, so set dynamically via a [capability](#dynamic-tool-choice-via-capabilities)or use[direct model requests](/docs/ai/core-concepts/direct); raises an error when set statically in`agent.run()`. | 
| `['tool_a', ...]` | Restrict to specific tools by name. Excludes output tools — same dynamic/direct requirement as `'required'`. | 
| `ToolOrOutput``(function_tools=['...'])` | Restrict function tools while auto-including all output tools. | 

```
from pydantic_ai import Agent
from pydantic_ai.models.test import TestModel
from pydantic_ai.settings import ToolOrOutput
agent = Agent(TestModel())
@agent.tool_plain
def get_weather(city: str) -> str:
    return f'Sunny in {city}'
@agent.tool_plain
def get_time(city: str) -> str:
    return f'12:00 in {city}'
# Pass tool_choice via model_settings
result = agent.run_sync('Hello', model_settings={'tool_choice': 'none'})
# Use ToolOrOutput to restrict to specific function tools while allowing output
result = agent.run_sync(
    'Hello', model_settings={'tool_choice': ToolOrOutput(function_tools=['get_weather'])}
)
```
`tool_choice='required'` and `['tool_a', ...]` exclude output tools, so setting either one *statically* would force a tool call on every step and leave the agent unable to produce a final response. `agent.run()` raises a `UserError` when it detects these values on the static baseline (the `model_settings` argument of `Agent.run`, the agent’s own `model_settings`, or the underlying model’s defaults).

To vary `tool_choice` *per step* — for example, to force a specific tool on the first step and then let the model decide — return a callable from a capability’s [ get_model_settings](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model_settings). The callable receives a 

[with full access to](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext``ctx.messages` and `ctx.run_step`, so it can inspect what has already happened in the run and adapt.Because capability-supplied settings are resolved per step, the callable’s returned `tool_choice` is trusted to change across steps and is not rejected by the baseline validator. For a single model request without an agent loop, use [ pydantic_ai.direct.model_request](/docs/ai/api/pydantic-ai/direct/#pydantic_ai.direct.model_request) instead.

All providers support `'auto'` and `'none'`. Key differences for other options:

| Provider | `'required'` | Specific tools | Notes | 
|---|---|---|---|
| OpenAI | ✓ | ✓ | Full support | 
| Anthropic | ⚠️ | ⚠️ | Not supported with thinking enabled | 
| ✓ | ✓ | ||
| Bedrock | ✓ | Single only | Multiple tools fall back to ‘any’ mode | 
| Groq/HuggingFace | ✓ | Single only | Multiple tools fall back to ‘required’ mode | 
| Mistral | ✓ | ✓ | Maps `'required'`to`'any'`mode | 
| xAI | ✓ | ✓ | Some models may not support forcing; falls back to ‘auto’ | 

Restricting the available tool set via `tool_choice` can invalidate provider prompt caches because most provider APIs cache on the full tools array. Pydantic AI restricts the tool set in two ways:

- **API-level filtering**(cache-preserving): the full tools array is sent and the provider is told to only allow a subset. Used by OpenAI Responses (- `allowed_tools`), Google (- `allowed_function_names`), and Bedrock when forcing a single tool.
- **Client-side filtering**(breaks cache): the tools array is trimmed before the request. Used when the provider API has no native filter for the given case.

The table below covers the cases where Pydantic AI must filter client-side and therefore breaks cache:

| Provider | Cache-breaking case | 
|---|---|
| Anthropic | `tool_choice`is a list of multiple tools, OR a single tool with thinking enabled | 
| OpenAI Chat | `tool_choice`is a list of multiple tools, OR a single tool on a model that doesn’t support forcing | 
| Bedrock | `tool_choice`is a list of multiple tools, OR a single tool with thinking enabled or on a model that doesn’t support forcing | 
| Groq / HuggingFace | `tool_choice`is a list of multiple tools | 
| Mistral | `tool_choice`is a list (any size) — the API doesn’t accept specific tool names | 
| xAI | `tool_choice`is a list of multiple tools, OR a single tool on a model that doesn’t support forcing | 
| OpenAI Responses | Never — `allowed_tools`handles all cases natively | 
| Never — `allowed_function_names`handles all cases natively | 

If preserving cache hits matters, prefer providers/cases marked “Never”, or use `ToolOrOutput` (which keeps the full set) instead of a restrictive list.

When a tool is executed, its arguments (provided by the LLM) are first validated against the function’s signature using Pydantic (with optional [validation context](/docs/ai/core-concepts/output#validation-context)). If validation fails (e.g., due to incorrect types or missing required arguments), a `ValidationError` is raised, and the framework automatically generates a [ RetryPromptPart](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.RetryPromptPart) containing the validation details. This prompt is sent back to the LLM, informing it of the error and allowing it to correct the parameters and retry the tool call.

Beyond automatic validation errors, the tool’s own internal logic can also explicitly request a retry by raising the [ ModelRetry](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) exception. This is useful for situations where the parameters were technically valid, but an issue occurred during execution (like a transient network error, or the tool determining the initial attempt needs modification).

```
from pydantic_ai import ModelRetry
def my_flaky_tool(query: str) -> str:
    if query == 'bad':
        # Tell the LLM the query was bad and it should try again
        raise ModelRetry("The query 'bad' is not allowed. Please provide a different query.")
    # ... process query ...
    return 'Success!'
```
Raising `ModelRetry` also generates a `RetryPromptPart` containing the exception message, which is sent back to the LLM to guide its next attempt. Both `ValidationError` and `ModelRetry` respect the configured retry limit — set per-tool via [ Tool(max_retries=N)](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) (or 

`@agent.tool(retries=N)`), per-toolset via [, or agent-wide via](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset)

`FunctionToolset(max_retries=N)`[, applied in that order of precedence.](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__)

`Agent(retries={'tools': N})`Tool retries are tracked **per tool**: every function tool has its own counter, with no global ‘tool call’ budget shared across the run. When a tool raises `ModelRetry` or its arguments fail validation, only that tool’s counter advances. Inside a tool function, [ ctx.max_retries](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.max_retries) reflects that tool’s enforcement limit and 

[is that tool’s own counter. When a tool exhausts its counter, the run raises](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.retry)

`ctx.retry`[with message](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UnexpectedModelBehavior)

`UnexpectedModelBehavior``'Tool {name!r} exceeded max retries count of {N}'`. User-provided toolsets inherit `Agent(retries={'tools': ...})` as their default when no per-toolset value is set.You can set a timeout for tool execution to prevent tools from running indefinitely. If a tool exceeds its timeout, it is treated as a failure and a retry prompt is sent to the model (counting towards the retry limit).

```
import asyncio
from pydantic_ai import Agent
# Set a default timeout for all tools on the agent
agent = Agent('test', tool_timeout=30)
@agent.tool_plain
async def slow_tool() -> str:
    """This tool will use the agent's default timeout (30 seconds)."""
    await asyncio.sleep(10)
    return 'Done'
@agent.tool_plain(timeout=5)
async def fast_tool() -> str:
    """This tool has its own timeout (5 seconds) that overrides the agent default."""
    await asyncio.sleep(1)
    return 'Done'
```
- **Agent-level timeout**: Set- `tool_timeout`on the- `Agent`
- **Per-tool timeout**: Set- `timeout`on individual tools via- `@agent.tool`- `@agent.tool_plain`- `Tool`

When a timeout occurs, the tool is considered to have failed and the model receives a retry prompt with the message `"Timed out after {timeout} seconds."`. This counts towards the tool’s retry limit just like validation errors or explicit [ ModelRetry](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) exceptions.

The `args_validator` parameter lets you define custom validation that runs after Pydantic schema validation but before the tool executes. This is useful for business logic validation, cross-field validation, or validating arguments before requesting [human approval](/docs/ai/tools-toolsets/deferred-tools) for deferred tools.

The validator receives [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) as its first argument, followed by the same parameters as the tool function. Return 

`None` on success, or raise [on failure.](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry)

`ModelRetry`*(This example is complete, it can be run “as is”)*

When validation fails, the error message is sent back to the LLM as a retry prompt. This respects the `retries` setting on the tool. For [deferred tools](/docs/ai/tools-toolsets/deferred-tools), validation runs at deferral time — only tool calls with valid arguments are deferred, while failed validation triggers a retry just like regular tools.

The `args_validator` parameter is available on [ @agent.tool](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool), 

[,](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain)

`@agent.tool_plain`[,](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool)

`Tool`[, and](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool.from_schema)

`Tool.from_schema`[. Validators can be sync or async functions.](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset)

`FunctionToolset`The validation result is exposed via the `args_valid` field on [ FunctionToolCallEvent](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.FunctionToolCallEvent). This reflects all validation — both schema validation and custom 

`args_validator` validation (if configured): `True` means all validation passed, `False` means validation failed, and `None` means validation was not performed (e.g. tool calls skipped due to the `'early'` end strategy, or deferred tool calls resolved without execution).When a model returns multiple tool calls in one response, Pydantic AI schedules them concurrently using `asyncio.create_task`, executing them in the order the model emitted them.

To stop a specific tool from overlapping with others, mark it `sequential=True` — it then acts as a barrier: tools the model emitted before it finish first, it runs alone, and tools emitted after it start only once it finishes.

You can pass the [ sequential](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition.sequential) flag when registering any function tool, and the same barrier is available for 

[output tools](/docs/ai/core-concepts/output#tool-output)via

[(see](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput)

`ToolOutput(sequential=True)`[Controlling output tool parallelism](/docs/ai/core-concepts/output#controlling-output-tool-parallelism)). To run an entire run’s tools serially regardless of which tools were called, wrap the run in the

[context manager, or set](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.parallel_tool_call_execution_mode)

`with agent.parallel_tool_call_execution_mode('sequential')``parallel_tool_calls=False` on the [model settings](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings).

Async functions are run on the event loop, while sync functions are offloaded to threads. To get the best performance, *always* use an async function *unless* you’re doing blocking I/O (and there’s no way to use a non-blocking library instead) or CPU-bound work (like `numpy` or `scikit-learn` operations), so that simple functions are not offloaded to threads unnecessarily.

By default, sync functions are offloaded to threads using `anyio.to_thread.run_sync`, which creates ephemeral threads on demand. In long-running servers (e.g. FastAPI), these threads can accumulate under sustained traffic, leading to memory growth.

To control thread lifecycle, provide a bounded [ ThreadPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.ThreadPoolExecutor) using the 

[capability (per-agent) or the](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ThreadExecutor)

`ThreadExecutor`[context manager (global):](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.using_thread_executor)

`Agent.using_thread_executor()````
from concurrent.futures import ThreadPoolExecutor
from contextlib import asynccontextmanager
from pydantic_ai import Agent
from pydantic_ai.capabilities import ThreadExecutor
# Per-agent: pass as a capability
executor = ThreadPoolExecutor(max_workers=16, thread_name_prefix='agent-worker')
agent = Agent('openai:gpt-5.2', capabilities=[ThreadExecutor(executor)])
# Global: wrap your server lifespan
@asynccontextmanager
async def lifespan(app):
    executor = ThreadPoolExecutor(max_workers=16)
    with Agent.using_thread_executor(executor):
        yield
    executor.shutdown(wait=True)
```
When a model calls an [output tool](/docs/ai/core-concepts/output#tool-output) in parallel with other tools, the agent’s [ end_strategy](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.end_strategy) parameter controls how these tool calls are executed.
The default 

`'graceful'` strategy ensures all function tools are executed even after a final result is found, while skipping remaining output tools. The `'exhaustive'` strategy goes further and also executes all output tools. Both are useful when tools have side effects (like logging, sending notifications, or updating metrics) that should always execute.For more information on how `end_strategy` works with both function tools and output tools, see [Parallel Output Tool Calls](/docs/ai/core-concepts/output#parallel-output-tool-calls).

Agents with many tools (e.g. [MCP servers](/docs/ai/mcp/client) exposing dozens of endpoints) can spend a lot of input tokens on tool definitions before any work happens, and tool selection accuracy noticeably degrades past ~30–50 available tools. Marking tools for deferred loading hides them from the model’s initial context; the model discovers hidden tools by keyword when it needs them.

For workflow *bundles* — instructions, tools, model settings, and hooks that travel together — see [on-demand capabilities](/docs/ai/core-concepts/capabilities#on-demand-capabilities), which build on the same machinery but disclose at the bundle level rather than the individual-tool level.

Reach for it when:

- the agent exposes ~10+ tools or more than ~10k tokens of tool definitions
- tools cover distinct domains (e.g. multiple MCP servers) and only a subset is relevant per request
- the toolset is growing and you want headroom

Skip it when you have a small, hot toolset where every tool is used most turns — deferring everything would just add a discovery round-trip for no benefit. As a rule of thumb, keep your handful of most-used tools eagerly loaded; defer the long tail.

To opt in, set `defer_loading=True` on individual [ Tool](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) / 

[/](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool)

`@agent.tool`[registrations, or use](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain)

`@agent.tool_plain`[on a whole toolset (including](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.defer_loading)

`.defer_loading()`[) — pass a list of tool names to hide specific ones, or](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset)

`MCPToolset``None` to hide all.Once deferred tools exist, search is handled by the auto-injected [ ToolSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch) capability:

- **Native provider search**on supporting models (Anthropic Sonnet 4.5+, Opus 4.5+, Haiku 4.5+ via- [BM25/regex](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool); OpenAI Responses on GPT-5.4+). Standalone deferred tools are sent to the provider with- `defer_loading`on the wire and the provider manages their visibility. Tools owned by on-demand capabilities use client-executed local search on native-supporting providers, because provider-side search cannot enforce capability gating before- `load_capability`succeeds.
- **Custom callable**via- `ToolSearch(strategy=...)`- `tool_reference`blocks, OpenAI- `execution='client'`) where supported so the model sees a tool-search call rather than a regular function tool.
- **Local fallback**on every other model: a- `search_tools`function tool matches keywords against tool names and descriptions.

Pydantic AI prefers native search whenever available because the discovery exchange happens append-only (a `tool_search_call` + `tool_search_output` pair) — the deferred tools never enter the prompt prefix, so prompt caching is preserved across rounds. The local fallback, by contrast, flips each discovered tool’s `defer_loading=False` between rounds, which changes the tool-definition prefix and invalidates the cached request prefix on every discovery turn.

Runs that include tools owned by [on-demand capabilities](/docs/ai/core-concepts/capabilities#on-demand-capabilities) trade hosted-search quality for capability gating and cache stability on native-supporting providers: deferred function tools are searched by Pydantic AI through the provider’s client-executed native surface, so each `load_capability` reveal can keep the prompt-cache prefix warm without exposing tools from unloaded capabilities. Runs with only standalone deferred tools keep using the provider’s hosted search.

For the model to find tools well, give them descriptive names with consistent prefixes (`github_*`, `slack_*`, `mortgage_*`) and put the keywords a user might search for in the tool’s description. A search returns a handful of matches at a time, so the model may iterate (search → discover → call → search again) — instructions can nudge it: “Search by topic when you don’t see a tool you need.”

For MCP servers, use [ .defer_loading()](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.defer_loading) to hide all tools behind search:

Pass an explicit [ ToolSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch) capability to control the strategy or provide a custom search function:

Available strategy values:

| `strategy` | Algorithm | Behavior | 
|---|---|---|
| `None`(default) | Provider’s native algorithm where available, else local keyword matching | Anthropic native BM25 on Sonnet 4.5+/Opus 4.5+/Haiku 4.5+, OpenAI server-executed `tool_search`on GPT-5.4+, local keyword matching elsewhere. | 
| `'keywords'` | Local keyword-overlap | The keyword algorithm runs on our side, but the wire shape adapts: client-executed native (Anthropic, OpenAI) where supported so the prompt cache stays warm, regular `search_tools`function tool elsewhere. | 
| `'bm25'`/`'regex'` | Anthropic native | Server-executed by Anthropic. The request fails on other providers (OpenAI, Google, etc.) rather than silently substituting a different algorithm. | 
| Callable `(ctx, queries, tools) -> names` | User-defined | Same execution-mode handling as `'keywords'`: client-executed native on supporting providers, local`search_tools`function tool elsewhere. | 

The execution mode (server-executed, client-executed-native, or local fallback) is auto-derived from the chosen algorithm and the current provider — users don’t pick it directly. Native execution is preferred whenever available because it keeps the model-facing tool list stable across discovery rounds, which preserves Anthropic and OpenAI prompt caching.

To force the local `keywords` algorithm on a provider that natively supports tool search, override [ ModelProfile.supported_native_tools](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.ModelProfile.supported_native_tools) to exclude 

`ToolSearchTool` — the capability then falls through to the local `search_tools` function tool.See [ ToolDefinition.defer_loading](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition.defer_loading) and 

[Deferred Loading](/docs/ai/tools-toolsets/toolsets#deferred-loading)for more details.

- [Function Tools](/docs/ai/tools-toolsets/tools)- Basic tool concepts and registration
- [Toolsets](/docs/ai/tools-toolsets/toolsets)- Managing collections of tools
- [Deferred Tools](/docs/ai/tools-toolsets/deferred-tools)- Tools requiring approval or external execution
- [Third-Party Tools](/docs/ai/tools-toolsets/third-party-tools)- Integrations with external tool libraries

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/tools-advanced
