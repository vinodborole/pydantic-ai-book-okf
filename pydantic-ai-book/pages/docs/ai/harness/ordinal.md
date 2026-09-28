---
type: Web Page
title: Ordinal | Pydantic Docs
description: Connect a Pydantic AI agent to Ordinal's hosted MCP server to draft,
  schedule, and analyze social posts.
resource: https://pydantic.dev/docs/ai/harness/ordinal
timestamp: '2026-09-28T13:22:55.549191+00:00'
---

# Ordinal

`Ordinal` connects an agent to [Ordinal](https://www.tryordinal.com)’s hosted MCP server so it can draft, schedule, and analyze social posts in the signed-in user’s workspaces.

While Pydantic AI Harness is on 0.x releases, the API may change between minor releases; when it does, deprecation warnings and release-note migration guidance tell you (or your agent) exactly how to upgrade. See the [version policy](/docs/ai/harness/#version-policy).

Ordinal MCP needs the Pro plan or higher, the same as the REST API. You need an Ordinal account with access to at least one workspace. Ordinal MCP takes an OAuth access token; there is no API key to copy.

Ordinal issues these tokens only through OAuth. Your application gets one by running the standard MCP authorization flow: the server names its authorization server and supports dynamic client registration, so any MCP OAuth client library can sign the user in. Store the token it returns and pass it as `auth` or `ORDINAL_ACCESS_TOKEN`.

If you set up the old server at `https://app.tryordinal.com/api/mcp` with a workspace API key, remove it. Do not use the old and new servers together, because their duplicate tool names confuse the agent.

The second package installs the OpenAI provider the example uses. For another model, install that provider’s extra instead.

```
from pydantic_ai import Agent
from pydantic_ai_harness import Ordinal
agent = Agent('openai:gpt-5.6-sol', capabilities=[Ordinal()])
result = agent.run_sync('List my Ordinal workspaces')
print(result.output)
```
Set `ORDINAL_ACCESS_TOKEN` to an Ordinal access token, or pass `auth=` a token. On your own machine, `auth='oauth'` signs you in through the browser instead. To serve several users from one agent, pass a function instead (see [Per-user credentials](#per-user-credentials)). Then start by listing workspaces. Every other Ordinal tool needs a `workspaceSlug` from `ordinal_get_workspace_context`.

`auth` decides which Ordinal account each run uses:

| `auth` | Account used | 
|---|---|
| Not set, `None` , or`''` | `ORDINAL_ACCESS_TOKEN` . If that is not set either, creating the agent raises an error. | 
| An access token | That token, for every run. | 
| `'oauth'` | The account you sign in to through the browser. This only works on your own machine. | 
| A function | Called at the start of each run. The token it returns is used for that run. If it returns `None` or`''` , that run has no Ordinal tools. A function never uses`ORDINAL_ACCESS_TOKEN` , and must not return`'oauth'` . | 

A fixed token or `ORDINAL_ACCESS_TOKEN` suits a script or an agent on your own machine, where every run is the same account.

In an app where each user connects their own Ordinal account, one agent serves all of them, so the token cannot be fixed when the agent is created. Pass a function that reads the current user’s token from the run’s deps:

```
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext
from pydantic_ai_harness import Ordinal
@dataclass
class Deps:
    ordinal_token: str | None
def ordinal_token(ctx: RunContext[Deps]) -> str | None:
    return ctx.deps.ordinal_token
agent = Agent('openai:gpt-5.6-sol', deps_type=Deps, capabilities=[Ordinal(auth=ordinal_token)])
```
Each run connects as its own user, so concurrent runs never share an account.

Your app gets each user’s token, stores it, and refreshes it. For example, a “Connect Ordinal” button that runs the OAuth flow from [Before you start](#before-you-start) and saves the token to their account. Before each run, load it (this can be async) and put it in the deps; the function only reads it.

With durable execution such as Temporal, read the token from the run’s deps rather than from a global, since the function may run in another process. The capability’s `id` defaults to `ordinal`, so `defer_loading=True` works without one. To add more than one `Ordinal` to an agent, give each a distinct `id` and wrap them in [PrefixTools](/docs/ai/capabilities/prefix-tools/), since their tool names are the same; two that share an `id` but differ raise an error.

To filter tools or require approval in your application, wrap the toolset with the existing [toolset wrappers](/docs/ai/tools-toolsets/toolsets/). For example, this asks for approval before every tool call, which suits tools that publish or schedule posts:

```
from pydantic_ai import Agent
from pydantic_ai.messages import DeferredToolRequests
from pydantic_ai_harness import Ordinal
capability = Ordinal()
agent = Agent(
    'openai:gpt-5.6-sol',
    toolsets=[capability.get_toolset().approval_required()],
    output_type=[str, DeferredToolRequests],
)
```
Handle the approval requests with the [deferred tools workflow](/docs/ai/tools-toolsets/deferred-tools/). To cap the size of tool output, add [Tool Output Limits](/docs/ai/harness/tool-output-limits/).

Use `auth` in almost every case. Pass `client` only when you need control of the connection itself: your own FastMCP client or transport, for example one with a proxy or MCP handlers. The client then owns the URL and authentication, so passing `client` together with `auth` raises an error. `include_instructions=False` stops the server’s own instructions from reaching the agent.

A `client` is one connection shared by every run; see [Per-user credentials](#per-user-credentials) to connect each user separately.

Loading a YAML file also needs the `spec` extra:

```
# agent.yaml
model: openai:gpt-5.6-sol
capabilities:
  - Ordinal: {}
```
```
from pydantic_ai import Agent
from pydantic_ai_harness import Ordinal
agent = Agent.from_file('agent.yaml', custom_capability_types=[Ordinal])
```
Pass `custom_capability_types` so the loader can create `Ordinal` from the file.

**Bases:** `AbstractCapability[AgentDepsT]`

Let an agent draft, schedule, and analyze social posts in Ordinal.

Set `ORDINAL_ACCESS_TOKEN` or pass an Ordinal access token as `auth`. The agent can then reach every
workspace the user belongs to.

```
from pydantic_ai import Agent
from pydantic_ai_harness import Ordinal
agent = Agent('openai:gpt-5.6-sol', capabilities=[Ordinal()])
```
Stable capability and toolset ID, so `defer_loading=True` needs none.

One `Ordinal` is one connection to one account, like `StackOne`’s linked account. Two sharing this `id` are
one connection stated twice when they agree, and an error when they differ; give each its own `id` to keep both.

Routing description used when the capability is loaded on demand.

**Type:** [`str`](https://docs.python.org/3/builtins/stdtypes.html#str) | `None`**Default:** `_DEFAULT_DESCRIPTION`

An Ordinal access token, `'oauth'` to sign in through the browser locally, or a function of the run context that returns a token.

Unset, it uses `ORDINAL_ACCESS_TOKEN`. A function never does: if it returns `None` or `''`, that run has no Ordinal tools.

**Type:** [`str`](https://docs.python.org/3/builtins/stdtypes.html#str) | [`Callable`](https://docs.python.org/3/library/typing.html#typing.Callable)[[[`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)[`AgentDepsT`]], [`str`](https://docs.python.org/3/builtins/stdtypes.html#str) | [`None`](https://docs.python.org/3/builtins/constants.html#None)] | `None`**Default:** `field(default=None, repr=False)`

Pass the server’s own instructions to the agent.

**Type:** `bool`**Default:** `True`

Your own MCP client or transport, for full control of the connection. It cannot be combined with `auth`.

**Type:** `MCPToolsetClient` | `None`**Default:** `field(default=None, repr=False)`

`@classmethod`

```
def combine(
    cls,
    capabilities: Sequence[AbstractCapability[AgentDepsT]],
) -> AbstractCapability[AgentDepsT]
```
Two under one `id` are one connection stated twice; two that disagree raise rather than merge.

`AbstractCapability`[`AgentDepsT`]

```
def get_toolset() -> AbstractToolset[AgentDepsT]
```
Return the Ordinal MCP tools.

[`AbstractToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset)[`AgentDepsT`]

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/ordinal
