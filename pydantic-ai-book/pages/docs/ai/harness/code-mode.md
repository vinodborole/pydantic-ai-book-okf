---
type: Web Page
title: Code Mode | Pydantic Docs
description: Wrap an agent's tools into a single sandboxed run_code tool so the model
  orchestrates many calls in one Python program instead of many round-trips.
resource: https://pydantic.dev/docs/ai/harness/code-mode
timestamp: '2026-07-20T09:23:04.251034+00:00'
---

# Code Mode

`CodeMode` replaces individual tool calls with a single sandboxed Python execution environment. Instead of the model issuing one tool call per action, it writes a Python program that calls your tools as functions — with loops, conditionals, variables, and `asyncio.gather` — all inside a sandboxed [Monty](https://github.com/pydantic/monty) runtime.

Standard tool calling costs one model round-trip per tool call. An agent that needs to fetch 10 items and process each one makes 11+ model calls — slow, expensive, and heavy on context. The conversation history grows with every intermediate result, and everything runs sequentially unless the model deliberately batches parallel tool calls.

`CodeMode` wraps the agent’s tools into a single `run_code` tool. The model writes one Python snippet that orchestrates many tool calls locally: fan them out with `asyncio.gather`, filter and transform results in plain Python, and return only what matters.

| Standard tool calling | Code mode | 
|---|---|
| 1 model call per tool | 1 model call for N tools | 
| Sequential by default | Parallel via `asyncio.gather` | 
| No local computation | Filter, transform, aggregate in code | 
| Large conversation history | Compact — fewer messages | 

Code mode requires the Monty sandbox, available via the `codemode` extra (the `code-mode` extra is an equivalent alias):

Construct an `Agent` with `CodeMode()` in its `capabilities`, then register tools as usual. Every tool becomes callable from inside `run_code`:

```
from pydantic_ai import Agent
from pydantic_ai_harness import CodeMode
agent = Agent('anthropic:claude-sonnet-4-6', capabilities=[CodeMode()])
@agent.tool_plain
def get_weather(city: str) -> dict:
    """Get current weather for a city."""
    return {'city': city, 'temp_f': 72, 'condition': 'sunny'}
@agent.tool_plain
def convert_temp(fahrenheit: float) -> float:
    """Convert Fahrenheit to Celsius."""
    return round((fahrenheit - 32) * 5 / 9, 1)
result = agent.run_sync("What's the weather in Paris and Tokyo, in Celsius?")
print(result.output)
```
Inside a single `run_code` call, the model writes code like the following (illustrative — the exact code the model emits will vary):

```
import asyncio
paris, tokyo = await asyncio.gather(
    get_weather(city='Paris'),
    get_weather(city='Tokyo'),
)
paris_c = await convert_temp(fahrenheit=paris['temp_f'])
tokyo_c = await convert_temp(fahrenheit=tokyo['temp_f'])
{'paris': paris_c, 'tokyo': tokyo_c}
```
Both weather lookups run in parallel, the conversions run locally, and the whole thing collapses into one model round-trip instead of five.

By default, `CodeMode(tools='all')` sandboxes every tool. The `tools` field is a Pydantic AI `ToolSelector`, so you can control precisely which tools go through the sandbox. Tools that match the selector become callables inside `run_code`; non-matching tools stay visible to the model as regular tool calls.

```
from pydantic_ai_harness import CodeMode
# By name -- only these tools are available inside run_code
CodeMode(tools=['search', 'fetch'])
# By predicate -- (ctx, tool_def) -> bool | Awaitable[bool]
CodeMode(tools=lambda ctx, td: td.name != 'dangerous_tool')
# By metadata -- combine with SetToolMetadata or a toolset's .with_metadata()
CodeMode(tools={'code_mode': True})
```
Use metadata when the decision should travel with a tool or toolset, rather than with one `CodeMode` instance. This suits shared toolsets: the toolset author tags the tools that are safe and useful to call from generated code, and each agent opts into that tag with `CodeMode(tools={...})`.

`CodeMode(tools={'code_mode': True})` uses the standard Pydantic AI [ ToolSelector](/docs/ai/api/pydantic-ai/tools/) metadata form. A tool is sandboxed when its 

`ToolDefinition.metadata` contains all of the selector’s key-value pairs. Extra metadata on the tool is fine, and nested dictionaries are matched by deep inclusion.The common pattern is to tag an entire toolset with `.with_metadata(...)`:

```
from pydantic_ai import Agent
from pydantic_ai.toolsets import FunctionToolset
from pydantic_ai_harness import CodeMode
def search(query: str) -> str:
    """Search the web."""
    return f'results for {query}'
def fetch(url: str) -> str:
    """Fetch a URL."""
    return f'contents of {url}'
search_tools = FunctionToolset(tools=[search, fetch]).with_metadata(code_mode=True)
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    toolsets=[search_tools],
    capabilities=[CodeMode(tools={'code_mode': True})],
)
```
Here `search` and `fetch` are removed from the model-facing tool list and become callable functions inside `run_code`. Tools without `metadata['code_mode'] == True` stay visible as regular tool calls.

When you mark tools or whole toolsets `defer_loading=True` ([Tool Search](/docs/ai/tools-toolsets/tools-advanced/#tool-search)), `CodeMode` keeps them out of `run_code` while they’re undiscovered — they pass straight through, so Tool Search drives them as usual (sent on the wire with `defer_loading` on providers with native tool search; otherwise dropped until discovered, with a `search_tools` tool alongside `run_code`). Once the model discovers a tool it comes back with `defer_loading=False`, and from then on `CodeMode` folds it into `run_code` like any other tool, so it’s callable from generated code.

That fold-in grows `run_code`’s description, which invalidates the prompt-cache prefix once at the moment of discovery (turns with no discovery stay cache-warm). Two ways to avoid the bust:

- 
Pass `dynamic_catalog=True`to keep`run_code`’s description static across discoveries. The catalog of sandboxed-tool signatures moves into the agent instructions (as a dynamic`InstructionPart``ctx.enqueue``from pydantic_ai_harness import CodeMode CodeMode(dynamic_catalog=True)`This pays off when paired with Tool Search: the tool-definitions block stays byte-stable so the prefix cache survives discoveries, at the cost of a larger (but cache-friendly) system prompt. With a fixed toolset and no Tool Search, the default keeps the system prompt shorter and is the better choice. 
- 
To instead keep a Tool Search corpus fully native — never folded into `run_code`, but not callable from inside it — exclude it with a`tools`selector; corpus members carry`with_native`set to the managing native tool:`from pydantic_ai_harness import CodeMode CodeMode(tools=lambda ctx, td: td.with_native is None)`

The last expression in the snippet is automatically captured as the return value — the model does not need to `print()`. Reserve `print()` for supplementary logging: printed text is surfaced separately, wrapped alongside the last-expression result.

| Scenario | Return | 
|---|---|
| No print output | Last expression value | 
| With print output | `{'output': '<printed text>', 'result': <last expression>}` | 
| Multimodal content (e.g. images) | Returned natively for model processing | 

State persists between `run_code` calls within the same agent run — variables, imports, and function definitions carry over. Pass `restart: true` in the tool call to reset state.

Nested tool calls inside `run_code` produce their own spans when instrumented with [Logfire](https://pydantic.dev/logfire) or any OpenTelemetry backend — the easiest way to understand what code mode actually did, since each `run_code` span fans out into the tool calls the model issued from inside the sandbox. See the [Pydantic AI Logfire docs](/docs/ai/integrations/logfire/) for setup.

The `run_code` tool return also carries metadata with every nested call, keyed by call id:

```
from pydantic_ai import Agent
from pydantic_ai.messages import ToolReturnPart
from pydantic_ai_harness import CodeMode
agent = Agent('anthropic:claude-sonnet-4-6', capabilities=[CodeMode()])
@agent.tool_plain
def get_weather(city: str) -> dict:
    """Get current weather for a city."""
    return {'city': city, 'temp_f': 72}
result = agent.run_sync("What's the weather in Paris?")
for msg in result.all_messages():
    for part in msg.parts:
        if isinstance(part, ToolReturnPart) and part.tool_name == 'run_code':
            metadata = part.metadata or {}
            tool_calls = metadata['tool_calls']    # dict[str, ToolCallPart]
            tool_returns = metadata['tool_returns']  # dict[str, ToolReturnPart]
```
A representative run wires `CodeMode` up against an MCP server and a web search and asks it to find the most-discussed Hacker News story across three feeds, pull the comment thread and the submitter’s profile, and search the web for follow-up coverage. `CodeMode` collapses that into two `run_code` calls: the first fetches all three feeds in parallel via `asyncio.gather`, dedupes by id, filters by score, and ranks by comment count — in plain Python; the second batches the three follow-up calls (`hn_get_thread`, `hn_get_user`, `duckduckgo_search`) together.

** See the full Logfire trace ->** Each 

`run_code` span fans out into the tool calls the model issued from inside the sandbox.Sandboxed code runs with no access to the host’s files, environment, or clock. Two parameters grant it access — reach for them only when the agent’s task genuinely needs the host.

Both parameters are fixed when the capability is built, so construct `CodeMode` per request to scope host access to that request.

Reach for `mount` when the agent works with real files: analyzing a dataset you’ve dropped in a folder and writing a report back, editing a checkout, or processing a batch of documents. Sandboxed `pathlib` code reads and writes under the mounted path. (For environment variables or the clock, use `os_access` instead.)

```
from pydantic_ai import Agent
from pydantic_monty import MountDir
from pydantic_ai_harness import CodeMode
# The agent can read /work/data.csv and write /work/summary.md back to the host:
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    capabilities=[CodeMode(mount=MountDir('/work', '/tmp/agent-workspace', mode='read-write'))],
)
```
A `MountDir` defaults to copy-on-write `mode='overlay'`: the sandbox reads host files and sees its own writes, but those writes do **not** reach the host. Pass `mode='read-write'` to persist them, or `mode='read-only'` to forbid writes. `mount` also accepts a list of `MountDir` for multiple mount points.

Reach for `os_access` when the agent needs environment variables, the current date and time, or filesystem behavior you control. Hand it a ready-made OS implementation (`AbstractOS`), or a callback that decides each call — so you can inject just the secrets it needs, pin “now” for reproducible runs, or route file access to your own store.

```
from pydantic_ai import Agent
from pydantic_monty import OSAccess
from pydantic_ai_harness import CodeMode
# Give the agent a fixed set of environment values:
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    capabilities=[CodeMode(os_access=OSAccess(environ={'API_BASE': 'https://api.example.com'}))],
)
```
A callback receives each OS call and decides its fate:

```
from pydantic_ai import Agent
from pydantic_monty import NOT_HANDLED
from pydantic_ai_harness import CodeMode
allowed_env = {'API_KEY': 'sk-...'}
def my_os(fn, args, kwargs):
    if fn == 'os.getenv':
        # Answer the call: allow-listed keys resolve, every other key reads back
        # as None -- absent, exactly like a real unset variable.
        return allowed_env.get(args[0])
    # Refuse everything else: NOT_HANDLED makes the call fail in the sandbox.
    return NOT_HANDLED
agent = Agent('anthropic:claude-sonnet-4-6', capabilities=[CodeMode(os_access=my_os)])
```
Your callback’s return value decides the call’s fate, and the two outcomes are easy to confuse:

- **Return any value**— including- `None`,- `''`, or- `0`— and that becomes the result the sandbox sees.- `os.getenv`returning- `None`looks exactly like a normal unset variable, so the agent’s code keeps running. This is how you- *hide*something: answer with an empty value.
- **Return**and the call is treated as unsupported: it raises inside the sandbox and the model gets a retry. This- `NOT_HANDLED`- *refuses*a capability outright — use it to block, not to say “no value”. Returning- `NOT_HANDLED`for a key the agent reasonably expects will burn retries.

Code runs inside [Monty](https://github.com/pydantic/monty), a sandboxed Python subset. Key restrictions:

- No class definitions.
- No third-party imports. Allowed stdlib modules: `sys`,`typing`,`asyncio`,`math`,`json`,`re`,`datetime`,`os`,`pathlib`(each must be imported before use).
- No wall-clock or timing primitives by default: `asyncio.sleep`,`datetime.datetime.now()`,`datetime.date.today()`, and the`time`module.`datetime.datetime.now()`/`datetime.date.today()`become available with an`os_access`handler (above);`asyncio.sleep`and`time`never do.
- No `import *`.
- Filesystem I/O needs an `os_access`handler or a`mount`;`os.getenv`/`os.environ`need an`os_access`handler.
- Tools requiring approval or with deferred (`CallDeferred`) execution are sandboxed like any other tool; without a`HandleDeferredToolCalls`(or equivalent) capability on the agent to resolve them inline, calling one from`run_code`raises an error that surfaces to the model as a retry.

`CodeMode` works with Pydantic AI’s [agent spec](/docs/ai/core-concepts/agent-spec/) feature for defining agents in YAML or JSON:

```
# agent.yaml
model: anthropic:claude-sonnet-4-6
capabilities:
  - CodeMode: {}
```
```
from pydantic_ai import Agent
from pydantic_ai_harness import CodeMode
agent = Agent.from_file('agent.yaml', custom_capability_types=[CodeMode])
result = agent.run_sync('...')
print(result.output)
```
Pass `custom_capability_types` so the spec loader knows how to instantiate `CodeMode`. Arguments can be passed in the YAML too:

```
capabilities:
  - CodeMode:
      tools: ['search', 'fetch']
      max_retries: 5
```
- [Tool use via code](https://www.anthropic.com/engineering/code-execution-with-mcp)(Anthropic)
- [Code mode in production](https://blog.cloudflare.com/code-mode/)(Cloudflare)
- [Pydantic AI capabilities](/docs/ai/core-concepts/capabilities/)

**Bases:** `AbstractCapability[AgentDepsT]`

Capability that exposes selected tools as callables inside a `run_code` sandbox.

By default (`tools='all'`) every tool the agent has is wrapped behind a single
`run_code` tool — the model writes Python that calls them as functions instead
of issuing tool calls directly.

Pass a list of tool names or a callable predicate to `tools` to split the
toolset: matching tools become callables inside the sandbox, and the rest
stay visible to the model as normal tool calls.

```
from pydantic_ai import Agent
from pydantic_ai_harness import CodeMode
# Sandbox all tools
agent = Agent('openai:gpt-5', capabilities=[CodeMode()])
# Sandbox only specific tools
agent = Agent('openai:gpt-5', capabilities=[CodeMode(tools=['search', 'fetch'])])
```
By default, sandboxed code cannot touch the host — no filesystem, environment variables, or clock. Two parameters open it up:

- `mount`shares specific host directories: reach for it when the agent reads or writes real files.
- `os_access`routes the sandbox’s OS calls to a handler you provide: reach for it when the agent needs environment variables, the clock, or filesystem behavior you control.

Both expose the real host to model-written code, so grant only what the task needs.

```
from pydantic_monty import MountDir
agent = Agent('openai:gpt-5', capabilities=[CodeMode(mount=MountDir('/work', '/tmp/agent-work'))])
```
Which wrapped tools should be sandboxed inside `run_code`.

- `'all'`(default): every tool the agent has is sandboxed.
- `Sequence[str]`: only tools whose names are listed are sandboxed.
- Callable `(ctx, tool_def) -> bool | Awaitable[bool]`: tools where the callable returns`True`are sandboxed; the rest stay as native tool calls.

**Type:** `ToolSelector`[`AgentDepsT`] **Default:** `field(default='all')`

Maximum number of retries for the `run_code` tool (syntax errors count as retries).

**Type:** `int`**Default:** `3`

Give sandboxed code environment variables, the clock, and file I/O through a handler you provide; unset, they are unavailable.

**Type:** `CodeModeOS` | `None`**Default:** `None`

Host directories to expose to sandboxed `pathlib` code; each mount’s `mode` controls whether writes reach the host.

**Type:** `CodeModeMount` | `None`**Default:** `None`

Keep the `run_code` tool definition cache-stable as the sandboxed toolset grows.

By default the signatures of all sandboxed tools are rendered into `run_code`’s
description, which lives in the prompt-cache-keyed tool-definitions block. When the
toolset changes mid-run — e.g. [ ToolSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch)
reveals a new tool that then gets folded into 

`run_code` — the description changes and
busts the prefix cache from that point on.Set `dynamic_catalog=True` to instead:

- keep only the static base prose (sandbox restrictions, return-value contract) in
`run_code.description`, so the tool-definitions block stays byte-stable across discoveries;
- move the “available functions” catalog (TypedDict definitions + signatures) into
agent instructions as a dynamic
`InstructionPart`
- announce newly-discovered tools via a short
`SystemPromptPart``RunContext.enqueue`

This pays off when paired with [ ToolSearch](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.ToolSearch): the
tool-definitions cache survives discoveries at the cost of a larger (but
cache-friendly) system prompt. With a fixed toolset and no 

`ToolSearch`, the default
keeps the system prompt shorter and is the better choice.**Type:** `bool`**Default:** `False`

```
def get_ordering() -> CapabilityOrdering
```
CodeMode wraps around ToolSearch so that search_tools stays native.

`CapabilityOrdering`

`@async`

```
def for_run(ctx: RunContext[AgentDepsT]) -> CodeMode[AgentDepsT]
```
Return a fresh instance so concurrent runs don’t share `_announced_tools`.

`CodeMode`[`AgentDepsT`]

```
def get_wrapper_toolset(
    toolset: AbstractToolset[AgentDepsT],
) -> AbstractToolset[AgentDepsT] | None
```
Wrap the agent’s assembled toolset, splitting it into native + sandboxed subsets if needed.

[ AbstractToolset](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset)[

`AgentDepsT`] | 

`None``@async`

```
def after_tool_execute(
    ctx: RunContext[AgentDepsT],
    *,
    call: ToolCallPart,
    tool_def: ToolDefinition,
    args: ValidatedToolArgs,
    result: Any,
) -> Any
```
Announce newly-discovered tools from a local `search_tools` return.

Only active with `dynamic_catalog=True`. The native-search path is handled by
[ after_model_request](/docs/ai/harness/code-mode/#pydantic_ai_harness.CodeMode.after_model_request) instead
(server-side search emits a 

`NativeToolSearchReturnPart` rather than a regular tool
execute result).`@async`

```
def after_model_request(
    ctx: RunContext[AgentDepsT],
    *,
    request_context: ModelRequestContext,
    response: ModelResponse,
) -> ModelResponse
```
Announce newly-discovered tools from a native (server-side) tool-search return.

Only active with `dynamic_catalog=True`.

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/code-mode
