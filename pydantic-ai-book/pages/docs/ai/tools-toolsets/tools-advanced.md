---
type: Web Page
title: Advanced Tool Features | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/tools-advanced
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Advanced Tool Features

This page covers advanced features for function tools in Pydantic AI. For basic tool usage, see the [Function Tools](/docs/ai/tools-toolsets/tools) documentation.

Tools can return anything that Pydantic can serialize to JSON, as well as audio, video, image or document content depending on the types of [multi-modal input](/docs/ai/core-concepts/input) the model supports:

*(This example is complete, it can be run “as is”)*

Some models (e.g. Gemini) natively support semi-structured return values, while some expect text (OpenAI) but seem to be just as good at extracting meaning from the data. If a Python object is returned and the model expects a string, the value will be serialized to JSON.

For scenarios where you need more control over both the tool’s return value and the content sent to the model, you can use [`ToolReturn`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolReturn). This is particularly useful when you want to:

- Separate the structured return value from additional content sent to the model
- Explicitly send content as a separate user message (rather than in the tool result)
- Include additional metadata that shouldn’t be sent to the LLM

Here’s an example of a computer automation tool that captures screenshots and provides visual feedback:

- **`return_value`** : The actual return value used in the tool response. This is what gets serialized and sent back to the model as the tool’s result. Can include multimodal content directly (see[Tool Output](#function-tool-output) above).
- **`content`** : Content sent as a**separate user message** after the tool result. Use this when you explicitly want content to appear outside the tool result, or when combining structured return values with rich content.
- **`metadata`** : Optional metadata that your application can access but is not sent to the LLM. Useful for logging, debugging, or additional processing. Some other AI frameworks call this feature ‘artifacts’.

This separation allows you to provide rich context to the model while maintaining clean, structured return values for your application logic. For multimodal content that should be sent natively in the tool result (when supported by the model), return it directly from the tool function or include it in `return_value` (see [Tool Output](#function-tool-output) above).

If you have a function that lacks appropriate documentation (i.e. poorly named, no type information, poor docstring, use of *args or **kwargs and suchlike) then you can still turn it into a tool that can be effectively used by the agent with the [`Tool.from_schema`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool.from_schema) function. With this you provide the name, description, JSON schema, and whether the function takes a `RunContext` for the function directly:

```
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

Some providers support a *strict* mode for tool calls that constrains the model so its tool-call arguments always conform to the tool’s JSON schema. Rather than letting the model generate arguments freely and validating them after the fact, the provider restricts generation so that out-of-schema arguments aren’t produced in the first place. This is controlled by the `strict` flag, available on every tool registration mechanism ([`@agent.tool`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool), [`@agent.tool_plain`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain), [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), [`FunctionToolset.add_function`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset.add_function), etc.) and on [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition):

```
from pydantic import BaseModel
from pydantic_ai import Agent
class Reservation(BaseModel):
    restaurant: str
    party_size: int
    outdoor_seating: bool
    dietary_notes: list[str]
agent = Agent('openai:gpt-5')
@agent.tool_plain(strict=True)
def book_table(reservation: Reservation) -> str:
    return f'Booked a table for {reservation.party_size} at {reservation.restaurant}.'
```
Strict mode earns its keep on structured arguments like this: without it the model might emit `party_size` as a string, omit `outdoor_seating`, or invent an extra property — strict generation rules those out up front rather than relying on a validation retry.

Because strict mode guarantees the arguments match the schema exactly, not every schema can be represented under it: some providers require every property to be listed in `required` and objects to set `additionalProperties: false`. A schema that can’t be represented this way may be transformed lossily or have the flag ignored for that tool — which is why `strict=True` is best read as a request to *force* strict mode wherever the provider can honor it.

Pydantic AI translates the `strict` flag into a native schema-enforcement feature for **OpenAI**, **Anthropic**, **Google**, and **Bedrock** models; other providers ignore it. Each provider’s underlying feature differs:

| Provider | Behavior | 
|---|---|
| OpenAI | Strict tool definitions. Enabled automatically when the tool’s schema is strict-compatible; `strict=True` forces it. | 
| Anthropic | Strict tool definitions. Off unless you opt in with `strict=True` . | 
| Bedrock | Strict tool spec, on supported models. Off unless you opt in with `strict=True` . | 
| Google (Gemini) | Gemini’s [`VALIDATED` function-calling mode](https://ai.google.dev/gemini-api/docs/function-calling#function_calling_config) , which ensures the model adheres to the declared schema. On**Gemini 2.5 and newer it is enabled by default** —`VALIDATED` needs no schema changes, so it’s a free improvement — and you can opt out with`strict=False` . Gemini’s mode is request-wide: any function or output tool with`strict=False` keeps the whole request on`AUTO` .`VALIDATED` is a preview Gemini feature. | 

The `strict` flag is a `bool | None`:

- `True` — force strict mode wherever the provider supports it for the tool’s schema.
- `False` — never use strict mode for the tool. On Google, any tool (function or output) with`strict=False` keeps the whole request on`AUTO` rather than`VALIDATED` .
- `None` (**default** ) — decide per provider: OpenAI enables strict when the schema is strict-compatible; Google defaults to`VALIDATED` on supported models; Anthropic and Bedrock leave it off unless you explicitly opt in with`strict=True` .

To turn strict mode on for many tools at once, use [agent-wide dynamic tools](#prepare-tools) to set `strict=True` on each [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition).

Tools can optionally be defined with another function: `prepare`, which is called at each step of a run to
customize the definition of the tool passed to the model, or omit the tool completely from that step.

A `prepare` method can be registered via the `prepare` kwarg to any of the tool registration mechanisms:

- [`@agent.tool`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool) decorator
- [`@agent.tool_plain`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain) decorator
- [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) dataclass

The `prepare` method, should be of type [`ToolPrepareFunc`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolPrepareFunc), a function which takes [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and a pre-built [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition), and should either return that `ToolDefinition` with or without modifying it, return a new `ToolDefinition`, or return `None` to indicate this tools should not be registered for that step.

Here’s a simple `prepare` method that only includes the tool if the value of the dependency is `42`.

As with the previous example, we use [`TestModel`](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) to demonstrate the behavior without calling a real model.

*(This example is complete, it can be run “as is”)*

Here’s a more complex example where we change the description of the `name` parameter to based on the value of `deps`

For the sake of variation, we create this tool using the [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) dataclass.

*(This example is complete, it can be run “as is”)*

In addition to per-tool `prepare` methods, you can also define an agent-wide `prepare_tools` function. This function is called at each step of a run and allows you to filter or modify the list of all tool definitions available to the agent for that step. This is especially useful if you want to enable or disable multiple tools at once, or apply global logic based on the current context.

The `prepare_tools` function should be of type [`ToolsPrepareFunc`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolsPrepareFunc), which takes the [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) and a list of [`ToolDefinition`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition), and returns the tool definitions to expose for that step. Return the `tool_defs` argument to keep every tool as-is, or `[]` to expose no tools.

To modify output tools, you can set a `prepare_output_tools` function instead.

Here’s an example that makes all tools strict if the model is an OpenAI model:

*(This example is complete, it can be run “as is”)*

Here’s another example that conditionally filters out the tools by name if the dependency (`ctx.deps`) is `True`:

*(This example is complete, it can be run “as is”)*

You can use `prepare_tools` to:

- Dynamically enable or disable tools based on the current model, dependencies, or other context
- Modify tool definitions globally (e.g., set all tools to strict mode, change descriptions, etc.)

If both per-tool `prepare` and agent-wide `prepare_tools` are used, the per-tool `prepare` is applied first to each tool, and then `prepare_tools` is called with the resulting list of tool definitions.

The `tool_choice` setting in [`ModelSettings`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings) controls which tools the model can use during a request. This is useful for disabling tools, forcing tool use, or restricting which tools are available.

Pydantic AI distinguishes between **[function tools](/docs/ai/tools-toolsets/tools)** (tools you register via `@agent.tool`, [toolsets](/docs/ai/tools-toolsets/toolsets), or [MCP](/docs/ai/mcp/client)), and **output tools** (internal tools used for [structured output](/docs/ai/core-concepts/output#tool-output)).

| Value | Description | 
|---|---|
| `'auto'` (default) | Model decides whether to use tools. All tools available. | 
| `'none'` | Disable function tools. Model can respond with text or use output tools. | 
| `'required'` | Force the model to use a function tool. Excludes output tools, so set dynamically via a [capability](#dynamic-tool-choice-via-capabilities) or use[direct model requests](/docs/ai/core-concepts/direct) ; raises an error when set statically in`agent.run()` . | 
| `['tool_a', ...]` | Restrict to specific tools by name. Excludes output tools — same dynamic/direct requirement as `'required'` . | 
| [`ToolOrOutput`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ToolOrOutput)`(function_tools=['...'])` | Restrict function tools while auto-including all output tools. | 

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
`tool_choice='required'` and `['tool_a', ...]` exclude output tools, so setting either one *statically* would force a tool call on every step and leave the agent unable to produce a final response. `agent.run()` raises a `UserError` when it detects these values on the static baseline (the `model_settings` argument of [`Agent.run`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.run), the agent’s own `model_settings`, or the underlying model’s defaults).

To vary `tool_choice` *per step* — for example, to force a specific tool on the first step and then let the model decide — return a callable from a capability’s [`get_model_settings`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.get_model_settings). The callable receives a [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) with full access to `ctx.messages` and `ctx.run_step`, so it can inspect what has already happened in the run and adapt.

Because capability-supplied settings are resolved per step, the callable’s returned `tool_choice` is trusted to change across steps and is not rejected by the baseline validator. For a single model request without an agent loop, use [`pydantic_ai.direct.model_request`](/docs/ai/api/pydantic-ai/direct/#pydantic_ai.direct.model_request) instead.

All providers support `'auto'` and `'none'`. Key differences for other options:

| Provider | `'required'` | Specific tools | Notes | 
|---|---|---|---|
| OpenAI | ✓ | ✓ | Full support | 
| Anthropic | ⚠️ | ⚠️ | Not supported with thinking enabled | 
|  | ✓ | ✓ |  | 
| Bedrock | ✓ | Single only | Multiple tools fall back to ‘any’ mode | 
| Groq/HuggingFace | ✓ | Single only | Multiple tools fall back to ‘required’ mode | 
| Mistral | ✓ | ✓ | Maps `'required'` to`'any'` mode | 
| xAI | ✓ | ✓ | Some models may not support forcing; falls back to ‘auto’ | 

Restricting the available tool set via `tool_choice` can invalidate provider prompt caches because most provider APIs cache on the full tools array. Pydantic AI restricts the tool set in two ways:

- **API-level filtering** (cache-preserving): the full tools array is sent and the provider is told to only allow a subset. Used by OpenAI Responses (`allowed_tools` ), Google (`allowed_function_names` ), and Bedrock when forcing a single tool.
- **Client-side filtering** (breaks cache): the tools array is trimmed before the request. Used when the provider API has no native filter for the given case.

The table below covers the cases where Pydantic AI must filter client-side and therefore breaks cache:

| Provider | Cache-breaking case | 
|---|---|
| Anthropic | `tool_choice` is a list of multiple tools, OR a single tool with thinking enabled | 
| OpenAI Chat | `tool_choice` is a list of multiple tools, OR a single tool on a model that doesn’t support forcing | 
| Bedrock | `tool_choice` is a list of multiple tools, OR a single tool with thinking enabled or on a model that doesn’t support forcing | 
| Groq / HuggingFace | `tool_choice` is a list of multiple tools | 
| Mistral | `tool_choice` is a list (any size) — the API doesn’t accept specific tool names | 
| xAI | `tool_choice` is a list of multiple tools, OR a single tool on a model that doesn’t support forcing | 
| OpenAI Responses | Never — `allowed_tools` handles all cases natively | 
|  | Never — `allowed_function_names` handles all cases natively | 

If preserving cache hits matters, prefer providers/cases marked “Never”, or use `ToolOrOutput` (which keeps the full set) instead of a restrictive list.

When a tool is executed, its arguments (provided by the LLM) are first validated against the function’s signature using Pydantic (with optional [validation context](/docs/ai/core-concepts/output#validation-context)). If validation fails (e.g., due to incorrect types or missing required arguments), a `ValidationError` is raised, and the framework automatically generates a [`RetryPromptPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.RetryPromptPart) containing the validation details. This prompt is sent back to the LLM, informing it of the error and allowing it to correct the parameters and retry the tool call.

If a tool’s own logic cannot produce a normal result, choose the exception based on what you want the model to do next:

- Raise [`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) when the model should try the tool call again with corrected arguments or a different approach.
- Raise [`ToolFailed`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ToolFailed) when the tool call should be reported to the model as a failed result, without consuming the tool’s retry budget.

Any other exception propagates out of the agent run and is not sent back to the model.

Raising `ModelRetry` generates a [`RetryPromptPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.RetryPromptPart) containing the exception message. That prompt is sent back to the LLM so it can correct the tool call, choose another tool, or try a different approach.

```
from pydantic_ai import ModelRetry
def my_flaky_tool(query: str) -> str:
    if query == 'bad':
        # Tell the LLM the query was bad and it should try again
        raise ModelRetry("The query 'bad' is not allowed. Please provide a different query.")
    # ... process query ...
    return 'Success!'
```
Both `ValidationError` and `ModelRetry` respect the configured retry limit — set per-tool via [`Tool(max_retries=N)`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) (or `@agent.tool(retries=N)`), per-toolset via [`FunctionToolset(max_retries=N)`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset), or agent-wide via [`Agent(retries={'tools': N})`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__), applied in that order of precedence. The agent-wide default can also be overridden per run via `agent.run(retries={'tools': N})` (and `run_sync`/`run_stream`/`iter`, or for a block of runs via [`agent.override()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.override)); a per-run value replaces the agent-wide default at the bottom of the precedence chain, so explicit per-tool and per-toolset limits still win. A bare `int` at these run-time call sites overrides both budgets (matching construction) — pass a dict such as `retries={'tools': N}` or `retries={'output': N}` to change just one.

Tool retries are tracked **per tool**: every function tool has its own counter, with no global ‘tool call’ budget shared across the run. When a tool raises `ModelRetry` or its arguments fail validation, only that tool’s counter advances. Inside a tool function, [`ctx.max_retries`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.max_retries) reflects that tool’s enforcement limit and [`ctx.retry`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.retry) is that tool’s own counter. When a tool exhausts its counter, the run raises [`UnexpectedModelBehavior`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UnexpectedModelBehavior) with message `'Tool {name!r} exceeded max retries count of {N}. Consider raising the retry limit, or see the docs on tool retries: https://ai.pydantic.dev/tools-advanced/#tool-retries'`. User-provided toolsets inherit the agent-wide tool-retry default — or its per-run override — as their default when no per-toolset value is set.

Two independent budgets — the **tool** budget (per function/output tool) and the **output** budget (output validation) — each resolve through the same layered precedence. The first layer that sets a value wins; unset layers fall through to the next:

| Precedence (highest first) | How to set it | Budget it sets | 
|---|---|---|
| 1. Per-tool limit | `@agent.tool(retries=N)` /[`Tool(max_retries=N)`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) ;[`ToolOutput(max_retries=N)`](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput.max_retries) for an output tool | that one tool | 
| 2. Per-toolset limit | [`FunctionToolset(max_retries=N)`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset) or[`MCPToolset(max_retries=N)`](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset) | tools in that toolset | 
| 3. Override block | [`agent.override(retries=...)`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.override) | tool and/or output | 
| 4. Per-run argument | `agent.run(retries=...)` (and`run_sync` /`run_stream` /`iter` ) | tool and/or output | 
| 5. Per-run spec | `agent.run(spec={'retries': ...})` | tool and/or output | 
| 6. Agent-wide default | [`Agent(retries=...)`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) | tool and/or output | 
| 7. Built-in default | — | `1` | 

At layers 3–6, a bare `int` sets **both** budgets to that value, while an [`AgentRetries`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AgentRetries) dict sets only the keys it names (`{'tools': N}`, `{'output': N}`, or both). Layers 3–5 override the agent-wide default (layer 6) but never a more specific per-tool (layer 1) or per-toolset (layer 2) limit.

Not every tool failure is a correction request. When the call is complete but unsuccessful — the resource doesn’t exist, the operation isn’t supported, the upstream service returned a definitive error — you usually want the model to *see* the failed result and decide what to do next. Raise `ToolFailed` for this:

```
from pathlib import Path
from pydantic_ai import ToolFailed
def read_file(path: str) -> str:
    file_path = Path(path)
    if not file_path.is_file():
        raise ToolFailed(f'File not found: {path}')
    return file_path.read_text()
```
The exception message is recorded in message history as a [`ToolReturnPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ToolReturnPart) with `outcome='failed'`. Where the model API has a native error or failed-status field for tool results, Pydantic AI uses it. For APIs without a native error channel, the model-visible content is JSON-framed as `{"error": ...}` so the failure is still explicit. The failed outcome is preserved in Pydantic AI message history; protocol adapters may need their own carrier when that history is round-tripped, as described for [AG-UI](/docs/ai/integrations/ui/ag-ui#preserving-failed-tool-outcomes). The call is traced as an error in telemetry.

Unlike `ModelRetry`, `ToolFailed` does **not** consume the per-tool retry budget; bounding repeated failures is the job of [`UsageLimits`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.UsageLimits) at the run level — specifically [`request_limit`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.UsageLimits.request_limit), since [`tool_calls_limit`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.UsageLimits.tool_calls_limit) only counts successful tool invocations.

Rule of thumb: raise `ModelRetry` when you want the model to try again with corrections; raise `ToolFailed` when the call is done and the result is a failure. For MCP server tool errors, the same choice is available as the [`tool_error_behavior`](/docs/ai/mcp/client#tool-errors) configuration.

You can also raise `ModelRetry` or `ToolFailed` from tool validation and execution hooks. This is useful for converting third-party exceptions without repeating `try`/`except` in every tool; see [Error hooks](/docs/ai/core-concepts/hooks#error-hooks) and [Tool execution hooks](/docs/ai/core-concepts/hooks#tool-execution-hooks).

`ToolFailed` is handled for function tools, their `args_validator`, and tool validation or execution hooks. [Output functions](/docs/ai/core-concepts/output#output-functions) and [output validators](/docs/ai/core-concepts/output#output-validator-functions) use `ModelRetry` when the model should try again; there, `ToolFailed` is an ordinary exception that aborts the run unless an output-process error hook recovers from it.

You can set a timeout for tool execution to prevent tools from running indefinitely. If a tool exceeds its timeout, it is treated as a retryable failure and a retry prompt is sent to the model (counting towards the retry limit).

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
- **Agent-level timeout** : Set`tool_timeout` on the[`Agent`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent) to apply a default timeout to all tools.
- **Per-tool timeout** : Set`timeout` on individual tools via[`@agent.tool`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool) ,[`@agent.tool_plain`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain) , or the[`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) dataclass. This overrides the agent-level default.

When a timeout occurs, the tool is treated as a retryable failure and the model receives a retry prompt with the message `"Timed out after {timeout} seconds."`. This counts towards the tool’s retry limit just like validation errors or explicit [`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) exceptions.

The `args_validator` parameter lets you define custom validation that runs after Pydantic schema validation but before the tool executes. This is useful for business logic validation, cross-field validation, or validating arguments before requesting [human approval](/docs/ai/tools-toolsets/deferred-tools) for deferred tools.

The validator receives [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) as its first argument, followed by the same parameters as the tool function. Return `None` on success, raise [`ModelRetry`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelRetry) to ask the model to correct the arguments and try again, raise [`ToolFailed`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ToolFailed) to report a terminal failure the model should adapt to instead of retrying, or raise [`ApprovalRequired`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ApprovalRequired) / [`CallDeferred`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.CallDeferred) to [defer the call](/docs/ai/tools-toolsets/deferred-tools) (see below).

*(This example is complete, it can be run “as is”)*

When schema validation fails, or an `args_validator` raises `ModelRetry`, the error message is sent back to the LLM as a retry prompt (with instructions to try again) and respects the tool’s `retries` setting. When an `args_validator` raises `ToolFailed`, the model instead receives a failed tool result it should adapt to rather than retry, and the retry budget is left untouched. For [deferred tools](/docs/ai/tools-toolsets/deferred-tools), validation runs at deferral time — only tool calls with valid arguments are deferred.

A validator can defer the call itself, just like the tool function can — and it’s the better place to make that decision, since bad arguments are rejected before a human is asked to approve them. A validator that raises `ApprovalRequired` or `CallDeferred` doesn’t consume the retry budget either: the arguments were valid, so the deferral is a deliberate decision rather than a failure. The tool function is not executed, and the call joins the run’s other [deferred tool calls](/docs/ai/tools-toolsets/deferred-tools): it’s resolved inline by a [`HandleDeferredToolCalls`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.HandleDeferredToolCalls) handler if you have one, or surfaced in the run’s `DeferredToolRequests` output. Once the call is [approved](/docs/ai/tools-toolsets/deferred-tools#human-in-the-loop-tool-approval), the validator runs again — this time with [`RunContext.tool_call_approved`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.tool_call_approved) set to `True` — and the tool executes.

Here, an impossible amount is rejected outright while a large one is put in front of a human:

The model can fix an impossible amount itself, without a human ever seeing the call.

Only calls that made it past validation are put in front of a human.

*(This example is complete, it can be run “as is”)*

The `args_validator` parameter is available on [`@agent.tool`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool), [`@agent.tool_plain`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain), [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool), [`Tool.from_schema`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool.from_schema), and [`FunctionToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.FunctionToolset). Validators can be sync or async functions.

The validation result is exposed via the `args_valid` field on [`FunctionToolCallEvent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.FunctionToolCallEvent). This reflects all validation — both schema validation and custom `args_validator` validation (if configured): `True` means all validation passed, `False` means validation failed, and `None` means validation was not performed (e.g. tool calls skipped due to the `'early'` end strategy, or deferred tool calls resolved without execution).

When a model returns multiple tool calls in one response, Pydantic AI schedules them concurrently using `asyncio.create_task`, executing them in the order the model emitted them.

To stop a specific tool from overlapping with others, mark it `sequential=True` — it then acts as a barrier: tools the model emitted before it finish first, it runs alone, and tools emitted after it start only once it finishes.

You can pass the [`sequential`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition.sequential) flag when registering any function tool, and the same barrier is available for [output tools](/docs/ai/core-concepts/output#tool-output) via [`ToolOutput(sequential=True)`](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.ToolOutput) (see [Controlling output tool parallelism](/docs/ai/core-concepts/output#controlling-output-tool-parallelism)). To run an entire run’s tools serially regardless of which tools were called, wrap the run in the [`with agent.parallel_tool_call_execution_mode('sequential')`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.parallel_tool_call_execution_mode) context manager, or set `parallel_tool_calls=False` on the [model settings](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings).

Async functions are run on the event loop, while sync functions are offloaded to threads. To get the best performance, *always* use an async function *unless* you’re doing blocking I/O (and there’s no way to use a non-blocking library instead) or CPU-bound work (like `numpy` or `scikit-learn` operations), so that simple functions are not offloaded to threads unnecessarily.

By default, sync functions are offloaded to threads using `anyio.to_thread.run_sync`, which creates ephemeral threads on demand. In long-running servers (e.g. FastAPI), these threads can accumulate under sustained traffic, leading to memory growth.

To control thread lifecycle, provide a bounded [`ThreadPoolExecutor`](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.ThreadPoolExecutor) using the [`UseThreadExecutor`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.UseThreadExecutor) capability (per-agent) or the [`Agent.using_thread_executor()`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent.using_thread_executor) context manager (global):

```
from concurrent.futures import ThreadPoolExecutor
from contextlib import asynccontextmanager
from pydantic_ai import Agent
from pydantic_ai.capabilities import UseThreadExecutor
# Per-agent: pass as a capability
executor = ThreadPoolExecutor(max_workers=16, thread_name_prefix='agent-worker')
agent = Agent('openai:gpt-5.2', capabilities=[UseThreadExecutor(executor)])
# Global: wrap your server lifespan
@asynccontextmanager
async def lifespan(app):
    executor = ThreadPoolExecutor(max_workers=16)
    with Agent.using_thread_executor(executor):
        yield
    executor.shutdown(wait=True)
```
When a model produces a final result — an [output tool](/docs/ai/core-concepts/output#tool-output) call, or structured [native](/docs/ai/core-concepts/output#native-output)/[prompted](/docs/ai/core-concepts/output#prompted-output) or [image](/docs/ai/core-concepts/output#image-output) output — in parallel with other tools, the agent’s [`end_strategy`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.end_strategy) parameter controls how these tool calls are executed.
The default `'graceful'` strategy ensures all function tools are executed even after a final result is found, while skipping remaining output tools. The `'exhaustive'` strategy goes further and also executes all output tools. Both are useful when tools have side effects (like logging, sending notifications, or updating metrics) that should always execute.

For more information on how `end_strategy` works with function tools, output tools, and non-tool output, see [Tool calls alongside a final result](/docs/ai/core-concepts/output#parallel-output-tool-calls).

Agents with many tools (e.g. [MCP servers](/docs/ai/mcp/client) exposing dozens of endpoints) can spend a lot of input tokens on tool definitions before any work happens, and tool selection accuracy noticeably degrades past ~30–50 available tools. Marking tools for deferred loading hides them from the model’s initial context; the model discovers hidden tools by keyword when it needs them.

For workflow *bundles* — instructions, tools, model settings, and hooks that travel together — see [on-demand capabilities](/docs/ai/capabilities/on-demand), which build on the same machinery but disclose at the bundle level rather than the individual-tool level.

Reach for it when:

- the agent exposes ~10+ tools or more than ~10k tokens of tool definitions
- tools cover distinct domains (e.g. multiple MCP servers) and only a subset is relevant per request
- the toolset is growing and you want headroom

Skip it when you have a small, hot toolset where every tool is used most turns — deferring everything would just add a discovery round-trip for no benefit. As a rule of thumb, keep your handful of most-used tools eagerly loaded; defer the long tail.

To opt in, set `defer_loading=True` on individual [`Tool`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.Tool) / [`@agent.tool`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool) / [`@agent.tool_plain`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool_plain) registrations, or use [`.defer_loading()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.defer_loading) on a whole toolset (including [`MCPToolset`](/docs/ai/api/pydantic-ai/mcp/#pydantic_ai.mcp.MCPToolset)) — pass a list of tool names to hide specific ones, or `None` to hide all.

Once deferred tools exist, search is handled by the auto-injected [`ToolSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch) capability:

- **Native provider search** on supporting models (Anthropic Sonnet 4.5+, Opus 4.5+, Haiku 4.5+ via[BM25/regex](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool) ; OpenAI Responses on GPT-5.4+). Standalone deferred tools are sent to the provider with`defer_loading` on the wire and the provider manages their visibility. On Anthropic, tools owned by on-demand capabilities use`defer_loading` without advertising tool search because`load_capability` controls their visibility. OpenAI uses client-executed local search for those tools because its API requires`tool_search` whenever a tool has`defer_loading` .
- **Custom callable** via[`ToolSearch(strategy=...)`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch) — a user-supplied search function. Executed on our side, but routed through the provider’s client-executed native surface (Anthropic`tool_reference` blocks, OpenAI`execution='client'` ) where supported so the model sees a tool-search call rather than a regular function tool.
- **Local fallback** on every other model: a`search_tools` function tool matches keywords against tool names and descriptions.

Pydantic AI prefers native search whenever available because the discovery exchange happens append-only (a `tool_search_call` + `tool_search_output` pair) while each tool’s authored `defer_loading` value remains stable, so prompt caching is preserved across rounds. On the local fallback, revealed tools are tracked separately from their stable definitions and sent only once discovered.

Toolsets that aggregate or wrap deferred definitions can check visibility with [`ctx.is_tool_available(tool_def)`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.is_tool_available) inside `get_tools`. Pass the definition the toolset is holding; the name form depends on the current resolved [`ctx.tools`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.tools) snapshot and is intended for model-request hooks and tool execution.

Runs that include only tools owned by [on-demand capabilities](/docs/ai/capabilities/on-demand) do not advertise tool search on Anthropic: the application-driven `load_capability` exchange reveals those tools directly. OpenAI keeps using its client-executed native surface because deferred tools require `tool_search` there. If a run also includes standalone deferred tools, normal model-driven tool search remains available.

For the model to find tools well, give them descriptive names with consistent prefixes (`github_*`, `slack_*`, `mortgage_*`) and put the keywords a user might search for in the tool’s description. A search returns a handful of matches at a time, so the model may iterate (search → discover → call → search again) — instructions can nudge it: “Search by topic when you don’t see a tool you need.”

For MCP servers, use [`.defer_loading()`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset.defer_loading) to hide all tools behind search:

Pass an explicit [`ToolSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch) capability to control the strategy or provide a custom search function:

Available strategy values:

| `strategy` | Algorithm | Behavior | 
|---|---|---|
| `None` (default) | Provider’s native algorithm where available, else local keyword matching | Anthropic native BM25 on Sonnet 4.5+/Opus 4.5+/Haiku 4.5+, OpenAI server-executed `tool_search` on GPT-5.4+, local keyword matching elsewhere. | 
| `'keywords'` | Local keyword-overlap | The keyword algorithm runs on our side, but the wire shape adapts: client-executed native (Anthropic, OpenAI) where supported so the prompt cache stays warm, regular `search_tools` function tool elsewhere. | 
| `'bm25'` /`'regex'` | Anthropic native | Server-executed by Anthropic. The request fails on other providers (OpenAI, Google, etc.) rather than silently substituting a different algorithm. | 
| Callable `(ctx, queries, tools) -> names` | User-defined | Same execution-mode handling as `'keywords'` : client-executed native on supporting providers, local`search_tools` function tool elsewhere. | 

The execution mode (server-executed, client-executed-native, or local fallback) is auto-derived from the chosen algorithm and the current provider — users don’t pick it directly. Native execution is preferred whenever available because it keeps the model-facing tool list stable across discovery rounds, which preserves Anthropic and OpenAI prompt caching.

To force the local `keywords` algorithm on a provider that natively supports tool search, override [`ModelProfile.supported_native_tools`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.ModelProfile.supported_native_tools) to exclude `ToolSearchTool` — the capability then falls through to the local `search_tools` function tool.

See [`ToolDefinition.defer_loading`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.ToolDefinition.defer_loading) and [Deferred Loading](/docs/ai/tools-toolsets/toolsets#deferred-loading) for more details.

Prompt caching keys on a **stable prefix**: providers cache the longest unchanged run of tokens from the start of the request, in roughly the order tool definitions → system/instructions → message history. A change at any layer invalidates the cache for that layer and everything after it — so on most providers **changing, adding, removing, or reordering a tool definition invalidates the cache**, because tool definitions sit at the very front.

Tool search works on **every model**, but it only *preserves* the cache where the model supports native tool search — Anthropic Sonnet 4.5+, Opus 4.5+, and Haiku 4.5+, and OpenAI Responses on GPT-5.4+. There, discovery is [append-only](#tool-search) and the deferred tools never enter the prompt prefix, so an identical prefix is re-read from cache across discovery rounds. On every other model — including Google, and older Anthropic and OpenAI models — the local `search_tools` fallback reveals a discovered tool by adding it to the tools array, which **invalidates the cached prefix from the tool definitions onward on each discovery turn** (on Google, a stable `system_instruction` sits ahead of the tool block and can still be reused — see [Related caching controls](#related-caching-controls) below).

For [on-demand capabilities](/docs/ai/capabilities/on-demand#on-demand-capabilities), loading a capability that reveals no new tool definitions — instructions or model settings only — preserves the cache on every provider, even without native tool search. Revealing a deferred function tool (on a non-native model) or a native tool enters the tool-definitions prefix; so does a deferred `prepare_tools`/`prepare_output_tools` hook that rewrites tool definitions on load. See [Cache implications](/docs/ai/capabilities/on-demand#cache-implications) for the full breakdown.

For a genuinely open-ended tool universe, route everything through a single, stable tool. The harness [`CodeMode`](https://pydantic.dev/docs/ai/harness/code-mode/) capability collapses many tools into one `run_code` tool whose definition stays byte-stable; newly discovered tools are surfaced as callables inside the sandbox rather than as new tool schemas, keeping the tool-definitions prefix — and its cache — intact across discoveries.

With native tool search the deferred catalog never enters the cached prefix, so an identical request prefix is re-read from cache on the next turn — `cache_read_tokens` stays warm even as the model discovers new tools:

Change a single tool definition, and the whole prefix is re-created instead — the [same request with one tool’s description edited](https://logfire-us.pydantic.dev/public-trace/c3205dc9-6251-40fa-9d0a-ed647be9ba30?spanId=38ef4635ecaf3e0b) records no cache read.

- Restricting the *active* tools with[`tool_choice`](#tool-choice) can also invalidate the cache when Pydantic AI has to trim the array client-side — see[Prompt caching implications](#tool-choice-caching) for the per-provider breakdown and the cache-preserving alternatives (`allowed_tools` ,`allowed_function_names` ,`ToolOrOutput` ).
- To place explicit cache breakpoints on messages, use [`CachePoint`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.CachePoint) (honored by Anthropic, Bedrock, and OpenRouter). Anthropic’s tool, system, and instruction caching settings are documented under[Anthropic prompt caching](/docs/ai/models/anthropic#prompt-caching) .
- On providers that cache tool definitions at the front of the prefix — Anthropic, OpenAI, and xAI — editing a single tool’s description invalidates the cached prefix. Google’s *implicit* cache is prefix-based on a different layout (its`system_instruction` is a separate field ahead of the tool block), so a large stable system instruction can keep cache hits even when the tool list changes; an explicit[`CachedContent`](/docs/ai/models/google) instead fixes the tools as an immutable part of the cache by construction.

- [Function Tools](/docs/ai/tools-toolsets/tools) - Basic tool concepts and registration
- [Toolsets](/docs/ai/tools-toolsets/toolsets) - Managing collections of tools
- [Deferred Tools](/docs/ai/tools-toolsets/deferred-tools) - Tools requiring approval or external execution
- [Third-Party Tools](/docs/ai/tools-toolsets/third-party-tools) - Integrations with external tool libraries

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/tools-advanced
