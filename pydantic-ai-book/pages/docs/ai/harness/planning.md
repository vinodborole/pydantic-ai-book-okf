---
type: Web Page
title: Planning | Pydantic Docs
description: Give an agent a structured, self-updating task plan through a single
  write_plan tool, without ever invalidating the prompt cache.
resource: https://pydantic.dev/docs/ai/harness/planning
timestamp: '2026-07-20T09:23:04.251034+00:00'
---

# Planning

`Planning` gives the model a structured, self-updating task plan through a single `write_plan` tool — and surfaces the current plan back to the model every turn without ever invalidating the prompt cache.

The API may change between releases. Where practical, breaking changes ship with a deprecation warning.

Long agentic runs drift: the model loses track of what it set out to do and what’s left. The usual fix — keep a running plan and re-inject it into the system prompt each turn — invalidates the prompt cache. The system prompt sits at the front of the request, so every plan edit changes the cached prefix and forces the whole conversation to be re-processed at full token price.

`Planning` gives the model one tool, `write_plan`, that owns the plan (whole-plan replacement — pass the full list every call, no indices). The current plan is surfaced back to the model as an ephemeral reminder appended to the tail of each request, behind a cache breakpoint:

- The reminder is added in `wrap_model_request`, which runs*after*the durable history is persisted, so it reaches the model but is never written to`message_history`. No reminders accumulate across turns.
- A `CachePoint`is placed immediately*before*the reminder, so the cached prefix (tools + system + real conversation) stays byte-identical turn over turn. Only the reminder falls outside the cache.

So the plan stays current in the model’s view while the cached prefix is never invalidated; the only added cost is re-reading the reminder each turn.

Construct an `Agent` with `Planning()` in its `capabilities`. The `write_plan` tool is registered automatically, and the static usage guidance is added to the system prompt:

```
from pydantic_ai import Agent
from pydantic_ai_harness.planning import Planning
agent = Agent('anthropic:claude-sonnet-4-6', capabilities=[Planning()])
result = agent.run_sync('Refactor the auth module and add tests.')
print(result.output)
```
| Tool | Purpose | 
|---|---|
| `write_plan(items)` | Create or replace the full plan. The model passes the entire ordered list every time, including unchanged, completed, and cancelled steps. | 

Each item is a `content` string plus a `status` (`pending`, `in_progress`, `completed`, `cancelled`). The convention — stated in the guidance and noted in the tool’s reply — is to keep exactly one step `in_progress`.

There is no `get_plan` tool: the current plan is already in the model’s context via the tail reminder every turn.

Addressing steps by mutable integer index (insert/remove/reorder) is error-prone for both the code (index bookkeeping) and the model (indices it just saw can go stale within a turn). Restating the whole plan each call removes that: there are no indices to track, and a later call can’t corrupt partial state. For short plans the token cost is negligible.

The plan is never injected into the system prompt or instructions. Static usage guidance goes there (cache-stable); only the mutable plan rides the ephemeral tail reminder. Across turns:

- the durable history grows append-only and is replayed byte-identically, so the whole prefix is a cache hit;
- the reminder and its `CachePoint`live only in the per-request copy, so they can’t invalidate anything and aren’t persisted.

`CachePoint` is supported on Anthropic and Amazon Bedrock; on providers without prompt caching it’s simply ignored (nothing to bust).

```
from pydantic_ai_harness.planning import Planning
Planning(
    guidance=None,      # static system-prompt guidance; None = default, '' = omit
    cache_ttl='5m',     # TTL for the cache breakpoint before the reminder ('5m' | '1h')
)
```
- `guidance`— static planning guidance added to the system prompt. It is identical on every request, so it stays cache-stable. Leave it as- `None`for the built-in default, or set- `''`to omit guidance entirely.
- `cache_ttl`— TTL for the cache breakpoint placed before the plan reminder. One of- `'5m'`or- `'1h'`.

Plan state is per-run (a fresh, isolated plan each run via `for_run`), so it doesn’t live on the `Planning()` instance you construct. To see the final plan, read the most recent `write_plan` tool return from the run’s messages — its content is the rendered plan:

```
from pydantic_ai import Agent
from pydantic_ai.messages import ToolReturnPart
from pydantic_ai_harness.planning import Planning
agent = Agent('anthropic:claude-sonnet-4-6', capabilities=[Planning()])
result = agent.run_sync('Refactor the auth module and add tests.')
plans = [
    part.content
    for message in result.all_messages()
    for part in message.parts
    if isinstance(part, ToolReturnPart) and part.tool_name == 'write_plan'
]
latest_plan = plans[-1] if plans else None
print(latest_plan)
```
`Planning` contributes a single leaf toolset (`write_plan`), some static instructions, and a `wrap_model_request` hook. It does not wrap or intercept other toolsets, so it composes cleanly alongside other capabilities and your own tools in the same `Agent(..., capabilities=[...])`.

The tail reminder is only appended when the last message in the request is a `ModelRequest` and the plan is non-empty, so an empty plan adds nothing to the request. Because the reminder and its `CachePoint` live only in the per-request copy and never enter the durable history, `Planning` is safe with message-history replay and does not accumulate stale reminders across turns.

`Planning` works with Pydantic AI’s [agent spec](/docs/ai/core-concepts/agent-spec/) feature for defining agents in YAML or JSON:

```
# agent.yaml
model: anthropic:claude-sonnet-4-6
capabilities:
  - Planning: {}
```
```
from pydantic_ai import Agent
from pydantic_ai_harness.planning import Planning
agent = Agent.from_file('agent.yaml', custom_capability_types=[Planning])
result = agent.run_sync('...')
print(result.output)
```
Pass `custom_capability_types` so the spec loader knows how to instantiate `Planning`. Arguments can be passed in the YAML too:

```
capabilities:
  - Planning:
      cache_ttl: '1h'
```
- [Pydantic AI capabilities](/docs/ai/core-concepts/capabilities/)
- [Hooks](/docs/ai/core-concepts/hooks/)—- `wrap_model_request`is the ephemeral injection point used here
- [Anthropic prompt caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching)
- [Code Mode](/docs/ai/harness/code-mode)— another prompt-cache-aware harness capability

**Bases:** `AbstractCapability[AgentDepsT]`

Structured task planning that never invalidates the prompt cache.

The plan is owned by the model through a single `write_plan` tool. The
current plan is surfaced back as an *ephemeral* reminder appended to the tail
of each request (after the latest message), with a cache breakpoint placed
in front of it. Because the reminder always sits after the breakpoint and is
never written to the durable message history, the cached prefix stays
byte-identical across turns — only the small reminder is re-read each turn.

Static usage guidance goes into the system prompt via `get_instructions`,
which is cache-stable; the mutable plan is *never* injected there.

```
from pydantic_ai import Agent
from pydantic_ai_harness.planning import Planning
agent = Agent('anthropic:claude-sonnet-4-6', capabilities=[Planning()])
```
Static planning guidance for the system prompt. Cache-stable (identical every
request). Leave as `None` for the default, or set `''` to omit guidance entirely.

**Type:** [ str](https://docs.python.org/3/library/stdtypes.html#str) | 

`None`**Default:**

`None`TTL for the cache breakpoint placed before the plan reminder.

**Type:** [ Literal](https://docs.python.org/3/library/typing.html#typing.Literal)[‘5m’, ‘1h’] 

**Default:**

`'5m'``@async`

```
def for_run(ctx: RunContext[AgentDepsT]) -> Planning[AgentDepsT]
```
Return a fresh per-run instance with isolated plan state (config preserved).

`Planning`[`AgentDepsT`]

```
def get_instructions() -> AgentInstructions[AgentDepsT] | None
```
Static, cache-stable guidance on using the planning tool.

`AgentInstructions`[`AgentDepsT`] | `None`

```
def get_toolset() -> AgentToolset[AgentDepsT] | None
```
Toolset providing `write_plan` over this run’s plan state.

`AgentToolset`[`AgentDepsT`] | `None`

`@async`

```
def wrap_model_request(
    ctx: RunContext[AgentDepsT],
    *,
    request_context: ModelRequestContext,
    handler: WrapModelRequestHandler,
) -> ModelResponse
```
Append the current plan as an ephemeral tail reminder behind a cache breakpoint.

This runs *after* core has persisted the durable history, and the
per-request message list it mutates is never written back. So the
reminder and its `CachePoint` reach the model but never enter
`ctx.state.message_history` — the cached prefix stays byte-identical
across turns and no stale reminders accumulate. The `CachePoint` sits
before the reminder text, so the reminder falls outside the cached
region and cannot invalidate it.

`@classmethod`

```
def get_serialization_name(cls) -> str | None
```
Serialization name for agent-spec support.

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/planning
