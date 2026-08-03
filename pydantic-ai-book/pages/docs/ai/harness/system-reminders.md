---
type: Web Page
title: System Reminders | Pydantic Docs
description: Re-inject behavioral guidance mid-run -- on a cadence or reactively --
  to counter instruction fade, without invalidating the prompt cache.
resource: https://pydantic.dev/docs/ai/harness/system-reminders
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# System Reminders

`SystemReminders` re-states targeted behavioral guidance partway through a run — on a fixed cadence or reactively from a condition — to counter the instruction fade that sets in over many turns, without ever invalidating the prompt cache.

The API may change between releases. Where practical, breaking changes ship with a deprecation warning.

Long multi-turn runs suffer instruction fade: after many tool-use turns the model progressively ignores the guidance it was given at the start. A single start-of-session system prompt is not enough for extended work. The fix is to re-state targeted guidance mid-run — on a fixed cadence, or reactively when a condition is detected.

`SystemReminders` injects reminders on each model request, either statically (`Reminder`, on a cadence) or dynamically (a callable that reads the run context). Reminders are appended to the **tail** of the request as an ephemeral `UserPromptPart` behind a `CachePoint`:

- The injection runs *after* the durable history is persisted, so the reminder reaches the model but is never written to`message_history` . No reminders accumulate across turns.
- A `CachePoint` is placed immediately*before* the reminder, so the cached prefix (tools + system + real conversation) stays byte-identical turn over turn. Only the small reminder falls outside the cache.

Injecting into the system prompt (or any persisted part) instead would sit at the front of the request, so every reminder would bust the cached prefix and stale reminders would pile up in history. This capability avoids both.

Construct an `Agent` with `SystemReminders(...)` in its `capabilities`:

```
from pydantic_ai import Agent
from pydantic_ai_harness.system_reminders import SystemReminders, Reminder
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    capabilities=[
        SystemReminders(
            reminders=[Reminder('Stay focused on the original request.', interval=5)],
        )
    ],
)
result = agent.run_sync('Refactor the auth module and add tests.')
print(result.output)
```
A `Reminder` fires on a cadence within a run:

| Field | Purpose | 
|---|---|
| `content` | The reminder text. | 
| `interval` | Fire every N model requests ( `interval=3` fires on the 3rd, 6th, …). | 
| `first_after` | Request number of the first fire, then every `interval` after.`None` = first multiple of`interval` (plain modulo). | 
| `trigger` | Predicate over `RunContext` . When set, fires only when it returns`True`*and* the cadence matches. | 
| `max_fires` | Cap the number of fires per run. `None` = no limit. | 
| `tag` | Wrap the content in `<tag>\ncontent\n</tag>` . Defaults to`'system-reminder'` ; set`None` for raw content. | 

The default `tag='system-reminder'` wraps every reminder in `<system-reminder>...</system-reminder>`, following Claude Code’s convention so the model reads it as an out-of-band steering note rather than user text.

The `tag` wrapping applies only to static `Reminder` content. Dynamic callables (including `GoalReanchor` and `LLMReminder`) inject their returned text raw and own their own formatting.

A dynamic reminder is any callable `(RunContext) -> str | None` (sync or async), evaluated on every model request. Return a string to inject, or `None` to skip. This is the general seam for conditions that need run state — token budget, post-compaction, mode switches — without hardcoded detectors:

```
from pydantic_ai_harness.system_reminders import SystemReminders
SystemReminders(
    dynamic_reminders=[
        lambda ctx: 'Wrap up soon.' if ctx.run_step > 20 else None,
    ],
)
```
`GoalReanchor` re-states the run’s first user request as the anchor and asks the model to check its next action advances it. No model call, no dependencies:

```
from pydantic_ai_harness.system_reminders import SystemReminders, GoalReanchor
SystemReminders(dynamic_reminders=[GoalReanchor()])
```
`LLMReminder` has a model summarize a compact transcript (original goal + recent activity) into a short stay-on-task nudge. It requires an explicit `model` — there is no default model id — and falls back to `GoalReanchor` text on any error, so a failed generation never blocks the run:

```
from pydantic_ai_harness.system_reminders import SystemReminders, LLMReminder
SystemReminders(dynamic_reminders=[LLMReminder(model='anthropic:claude-haiku-4-5')])
```
Dynamic reminders have no cadence of their own — they run on every model request. `LLMReminder` therefore issues one extra model call per turn (its usage is threaded onto the parent run via `ctx.usage`, so it shows up in `result.usage()`). The nested call also runs under the parent’s `usage_limits` with one request held back for the model request it precedes, so the reminder cannot push a run past its `request_limit`; once the budget is that tight the generation is skipped and `GoalReanchor` text is used instead. Because the fallback is silent, a persistently misconfigured `model` (bad id, missing key) looks like normal operation. To bound the cost, gate it behind a cadence with an async wrapper:

```
_llm = LLMReminder(model='anthropic:claude-haiku-4-5')
async def every_tenth(ctx):
    return await _llm(ctx) if ctx.run_step % 10 == 0 else None
SystemReminders(dynamic_reminders=[every_tenth])
```
`LLMReminder` generates inside `wrap_model_request`, so under durable execution (Temporal, DBOS, Prefect) its model call runs in orchestration context rather than a durable step — non-deterministic on replay and not checkpointed, with errors falling back silently to `GoalReanchor`. For durable runs prefer `GoalReanchor` (no model call) or gate `LLMReminder` off.

```
from pydantic_ai_harness.system_reminders import SystemReminders, Reminder
SystemReminders(
    reminders=[Reminder('...', interval=5)],
    dynamic_reminders=[],       # callables evaluated every request
    cache_ttl='5m',             # TTL for the cache breakpoint before the reminder ('5m' | '1h')
    on_fire=None,               # optional callback invoked with each rendered reminder
)
```
Per-run state (the request counter and per-reminder fire counts) is isolated via `for_run`, so concurrent runs on the same agent never share fire state.

Reminders are never injected into the system prompt or instructions. They ride the ephemeral tail behind a `CachePoint`, so across turns:

- the durable history grows append-only and is replayed byte-identically, so the whole prefix stays eligible for a cache hit (subject to the provider’s cache TTL — a gap longer than `cache_ttl` expires the entry even under an unchanged prefix);
- the reminder and its `CachePoint` live only in the per-request copy, so they can’t invalidate anything and aren’t persisted.

`CachePoint` is supported on Anthropic, Amazon Bedrock (Converse API), and OpenRouter (Anthropic and Gemini models); on providers without prompt caching it’s simply ignored (nothing to bust). The reminder leads with its `CachePoint` only when the request already carries user content for the breakpoint to attach to — on a turn whose only tail content is the reminder (for example an `instructions`-only run’s first request), the reminder is injected without a breakpoint, since there is no prefix to protect.

- [Planning](/docs/ai/harness/planning) uses the same ephemeral-tail mechanism to surface the plan. Both compose in one agent: each appends its own tail part behind its own`CachePoint` , and neither is persisted. Note that each ephemeral-tail capability adds a cache breakpoint: Anthropic allows 4 (3 with automatic caching), and core trims the excess oldest-first, so stacking several tail-injecting capabilities alongside`anthropic_cache_instructions` /`anthropic_cache_tool_definitions` can evict an older breakpoint. Two capabilities plus the defaults stay within budget.
- Loop detection (detect-and-interrupt with a durable nudge) is a separate concern. `SystemReminders` is cadence/condition steering that stays ephemeral; a dynamic reminder can read loop state from your deps if you want to steer on it.

The tail reminder is only appended when the last message in the request is a `ModelRequest` and at least one reminder fires, so a turn where nothing fires adds nothing to the request. Provider-resume turns (where the request tail is a suspended `ModelResponse` that is echoed back verbatim) are skipped and do not consume a cadence slot.

`SystemReminders.get_serialization_name()` returns `None`: reminders take arbitrary callables, which cannot be serialized to an [agent spec](/docs/ai/core-concepts/agent-spec/).

- [Pydantic AI capabilities](/docs/ai/capabilities/overview/)
- [Hooks](/docs/ai/core-concepts/hooks/) —`wrap_model_request` is the ephemeral injection point used here
- [Anthropic prompt caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching)
- [Planning](/docs/ai/harness/planning) — another prompt-cache-aware harness capability

**Bases:** `AbstractCapability[AgentDepsT]`

Inject periodic or conditional reminders to counter instruction fade in long sessions.

Long multi-turn runs suffer instruction fade: after many tool-use turns the model
progressively ignores start-of-session guidance. `SystemReminders` re-injects targeted
guidance mid-run, either on a fixed cadence (`Reminder`) or reactively from a callable
(`dynamic_reminders`).

Cache safety is the design constraint. Reminders are appended to the *tail* of each
request as an ephemeral `UserPromptPart` behind a `CachePoint`, inside `wrap_model_request`
(which runs after core persists the durable history). So reminders reach the model but
never enter `message_history`: no stale reminders accumulate, and the cached prefix stays
byte-identical across turns — only the small reminder falls outside the cache. Injecting
into the system prompt or a persisted part instead would bust the cache prefix on every
fire and let reminders pile up.

```
from pydantic_ai import Agent
from pydantic_ai_harness.system_reminders import SystemReminders, Reminder
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    capabilities=[
        SystemReminders(
            reminders=[Reminder('Stay focused on the original request.', interval=5)],
        )
    ],
)
```
Static reminders injected on a cadence.

**Type:** [`Sequence`](https://docs.python.org/3/library/typing.html#typing.Sequence)[`Reminder`[`AgentDepsT`]] **Default:** `()`

Callables evaluated every model request; return text to inject or `None` to skip.

**Type:** [`Sequence`](https://docs.python.org/3/library/typing.html#typing.Sequence)[`DynamicReminder`[`AgentDepsT`] | `AsyncDynamicReminder`[`AgentDepsT`]] **Default:** `()`

TTL for the cache breakpoint placed before the tail reminder.

**Type:** [`Literal`](https://docs.python.org/3/library/typing.html#typing.Literal)[‘5m’, ‘1h’] **Default:** `'5m'`

Optional observability callback invoked with each rendered reminder as it fires.

**Type:** [`Callable`](https://docs.python.org/3/library/typing.html#typing.Callable)[[[`str`](https://docs.python.org/3/library/stdtypes.html#str)], [`None`](https://docs.python.org/3/library/constants.html#None)] | `None`**Default:** `None`

`@async`

```
def for_run(ctx: RunContext[AgentDepsT]) -> SystemReminders[AgentDepsT]
```
Return a fresh per-run instance with reset counters (config preserved).

`replace` builds a new instance whose generated `__init__` re-initializes the
`init=False` fields — `_request_count` back to `0` and `_fire_counts` to an empty
dict — so concurrent runs on the same agent never share fire state.

`SystemReminders`[`AgentDepsT`]

`@async`

```
def wrap_model_request(
    ctx: RunContext[AgentDepsT],
    *,
    request_context: ModelRequestContext,
    handler: WrapModelRequestHandler,
) -> ModelResponse
```
Append fired reminders to the request tail behind a cache breakpoint, then call the model.

Runs after core persists the durable history; the per-request message list mutated
here is never written back, so the reminder and its `CachePoint` reach the model but
never enter `ctx.state.message_history`.

`@classmethod`

```
def get_serialization_name(cls) -> str | None
```
Not spec-serializable: reminders take arbitrary callables.

**Bases:** `Generic[AgentDepsT]`

A static reminder injected on a cadence during an agent run.

The reminder text.

**Type:** `str`

Fire every N model requests within a run. `interval=3` fires on the 3rd, 6th, 9th, … request.

**Type:** `int`**Default:** `1`

Request number of the first fire. `None` (the default) fires on the first multiple of
`interval` (plain modulo). When set, the reminder fires at `first_after`, then every
`interval` requests after that.

**Type:** [`int`](https://docs.python.org/3/library/functions.html#int) | `None`**Default:** `None`

Optional predicate over the current `RunContext`. When set, the reminder fires only when
the trigger returns `True` *and* the cadence condition is met.

**Type:** [`Callable`](https://docs.python.org/3/library/typing.html#typing.Callable)[[[`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)[`AgentDepsT`]], [`bool`](https://docs.python.org/3/library/functions.html#bool)] | `None`**Default:** `None`

Maximum number of times this reminder may fire within a run. `None` means no limit.

**Type:** [`int`](https://docs.python.org/3/library/functions.html#int) | `None`**Default:** `None`

When set, wrap the content in an XML tag: `<tag>\ncontent\n</tag>`. Defaults to
`'system-reminder'` (Claude Code’s convention); set `None` to emit the raw content.

**Type:** [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `None`**Default:** `'system-reminder'`

**Bases:** `Generic[AgentDepsT]`

Zero-cost dynamic reminder that re-states the run’s first user request as the anchor.

No model call and no dependencies: it reads the first user message from `ctx.messages`
and asks the model to check its next action advances that goal. Falls back to a static
line when there is no user message yet. Add it to `SystemReminders.dynamic_reminders`.

**Bases:** `Generic[AgentDepsT]`

Dynamic reminder whose text a model generates from a compact transcript.

Opt-in and dependency-free (it uses `pydantic_ai.Agent`). `model` is required and has no
default — pass an explicit model. On any error it falls back to `GoalReanchor` text, so a
failed generation never blocks the run. Add it to `SystemReminders.dynamic_reminders`.

Like every dynamic reminder it is evaluated on every model request, so it issues one extra
model call per turn; its usage is threaded onto the parent run (`ctx.usage`) and it runs
under the parent’s `usage_limits` minus one reserved request, so the reminder cannot push a
run past its `request_limit`. Once the budget is that tight the generation is skipped and
`GoalReanchor` text is used instead. Gate it on a cadence (see the docs) if per-turn
generation is too costly.

The generation runs inside `wrap_model_request`, so under durable execution (Temporal,
DBOS, Prefect) it executes in orchestration context rather than a durable step: the model
call is non-deterministic on replay and is not checkpointed, and its errors fall back
silently to `GoalReanchor`. For durable runs prefer `GoalReanchor`, which makes no model
call, or gate `LLMReminder` off.

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/system-reminders
