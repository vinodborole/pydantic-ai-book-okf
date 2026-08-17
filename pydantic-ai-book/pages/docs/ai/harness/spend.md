---
type: Web Page
title: Spend | Pydantic Docs
description: Track what an agent costs and refuse the next request once a budget is
  spent, with windows longer than a run, per-tenant scopes, and a counter shared across
  worker processes.
resource: https://pydantic.dev/docs/ai/harness/spend
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Spend

Track what an agent costs, and stop it when a budget is gone.

While Pydantic AI Harness is on 0.x releases, the API may change between minor releases; when it does, deprecation warnings and release-note migration guidance tell you (or your agent) exactly how to upgrade. See the [version policy](/docs/ai/harness/#version-policy).

A loop that calls a model until a condition it never reaches will keep calling until something stops it. `UsageLimits` in Pydantic AI is that stop for one run: it caps tokens and requests, in token counts, for the duration of a single `run()`. What it does not cover is money, a period longer than one run, a per-tenant share of a shared allowance, or a counter that several worker processes agree on. A daily ceiling spread across a queue’s workers is exactly the case where each worker independently believes it has the whole budget.

Provider usage APIs do not close that gap. They are billing and observability pipelines: usage is aggregated after the fact and read by polling, so a number there moves only once the requests behind it have already been made. That is enough to reconcile a ledger and not enough to refuse the request a runaway loop is about to make.

`SpendLimits` prices every model response with [`ModelResponse.cost()`](https://pydantic.dev/docs/ai/api/messages/), adds it to each window you configure, and refuses the next request once a window is spent.

```
from decimal import Decimal
from pydantic_ai import Agent
from pydantic_ai_harness import SpendLimits
from pydantic_ai_harness.spend import Budget
agent = Agent(
    'openai:gpt-5.4',
    capabilities=[SpendLimits(budgets=[Budget(usd=Decimal('100'), window='day')])],
)
```
Past $100 in a UTC day, the next request raises `SpendLimitExceeded`.

A budget is a ceiling, a period, and optionally a partition. They compose, so several apply at once:

```
from decimal import Decimal
from pydantic_ai_harness import SpendLimits
from pydantic_ai_harness.spend import Budget
SpendLimits(
    budgets=[
        Budget(usd=Decimal('5'), window='run'),  # one runaway run
        Budget(usd=Decimal('100'), window='day'),  # the whole deployment, per day
        Budget(usd=Decimal('2000'), window='month', warn_at=0.8),
        Budget(usd=Decimal('10'), window='day', scope=lambda ctx: ctx.deps.tenant_id, name='tenant'),
    ]
)
```
| Field | Meaning | 
|---|---|
| `usd` /`tokens` | ceilings; set either, both, or neither | 
| `window` | `run` ,`conversation` ,`day` ,`month` ,`total` | 
| `scope` | derives a partition key from the run, so tenants count separately; typed against the agent’s `deps` | 
| `warn_at` | fraction past which `BudgetStatus.warning` is set; never blocks | 
| `name` | distinguishes budgets sharing a window and scope | 
| `retain` | how long the counter is kept after its last write; `'window default'` ,`'forever'` , or a`timedelta` | 

A window rolls over by producing a different store key rather than by resetting a counter, so a new day is simply a new key and nothing has to run at midnight. A `total` counter never expires. `run` and `conversation` buckets never roll over either, so expiry there hands back the ceiling rather than starting a new period — but each mints a key per run or per conversation, so they carry a long horizon (24 hours and 30 days) instead, past which the counter is dropped rather than kept forever. That default is a compromise, and it is visible: a conversation resumed past its horizon starts from zero again, so set `retain='forever'` where a conversation ceiling has to hold for as long as the conversation does, and clean the keys up some other way.

Budgets that share a `name`, `window`, and `scope` share one counter, which is how a single window carries both a USD and a token ceiling. The response is added to that counter once, not once per budget. Two budgets that share a `name` and `window` but declare *different* `scope` callables are refused at construction: they are different dimensions — per tenant and per user, say — and nothing stops the two returning the same string, which would merge them into one counter. Give them different names, or pass the same callable to both. Budgets that do share a counter must also agree on `retain`: one counter has one expiry, and the accrual writes whichever of them is listed first, so disagreeing would let declaration order decide when a `'forever'` ceiling rolls over.

`Budget` is generic in the agent’s dependency type, so a `scope` is checked against it: pass the capability to an `Agent` with a `deps_type` and a scope reaching for a field those deps do not have is a type error rather than an `AttributeError` on the first request.

**A budget with no ceiling is a counter.** It accumulates and reports and never refuses anything, which is how per-tenant accounting with no cap is expressed:

```
from pydantic_ai_harness import SpendLimits
from pydantic_ai_harness.spend import Budget
SpendLimits(budgets=[Budget(window='month', scope=lambda ctx: ctx.deps.tenant_id, name='chargeback')])
```
No request **starts** after a budget is exhausted.

Not: that spend stays under the ceiling. The request that crosses the line completes, and concurrent runs can each pass the check before any of them records anything. Three further gaps are worth knowing rather than discovering: a stream the caller abandons part-way never reaches the accounting hook, so its tokens are billed by the provider and invisible here; a capability that answers from a cache without calling a provider is charged the registry price for the response it returns; and a continuation chain (Anthropic `pause_turn`, OpenAI background mode) arrives at the hook as one merged response, which is what Pydantic AI counts as one request too, so its segments are priced on summed usage rather than one at a time — the difference only shows where pricing is tiered rather than linear. Treat this as a brake on a runaway loop, not as an accounting ledger; reconcile against the provider’s own numbers if you need the second thing.

```
from decimal import Decimal
from pydantic_ai_harness import SpendLimits
from pydantic_ai_harness.spend import Budget, SpendSnapshot
def show(snapshot: SpendSnapshot) -> None:
    print(f'{snapshot.model} cost ${snapshot.usd}')
    for status in snapshot.budgets:
        print(f'  {status.budget.name}: ${status.remaining_usd} left')
SpendLimits(budgets=[Budget(usd=Decimal('100'))], on_spend=show)
```
`on_spend` fires after every response, sync or async, with a `SpendSnapshot` — including one that `on_unpriced='raise'` is about to reject, since a report that skipped exactly the unpriced responses would be missing the ones worth knowing about. It carries the response’s `usage` unchanged, so cache reads and writes are available without this capability modelling them.

`status()` reads the same numbers without a run, which is what a cost display in a UI wants:

```
async def report(limits: SpendLimits[None]) -> None:
    for status in await limits.status(scope='acme'):
        print(status.budget.name, status.spent.usd, status.exhausted)
```
Without a run context, budgets on a `run` or `conversation` window are omitted, and so is a budget declaring a `scope` unless `scope=` names the partition to read. Pass `ctx` inside a run and every budget resolves.

Set `expose_tools=True` to give the agent a `get_spend` tool. It is off by default: a tool costs schema tokens on every request, and most applications want the number on a screen rather than in the model’s context.

The default store keeps counters in the process, which catches a runaway loop inside one worker and does nothing for a budget spread across a queue. `RedisSpendStore` is the shared counter:

```
from decimal import Decimal
from redis.asyncio import Redis
from pydantic_ai_harness import SpendLimits
from pydantic_ai_harness.spend import Budget, RedisSpendStore
store = RedisSpendStore(Redis.from_url('redis://localhost'))
limits = SpendLimits(budgets=[Budget(usd=Decimal('100'), window='day')], store=store)
```
It adds no dependency: `RedisClient` is a protocol of the two coroutines used, so any compatible client satisfies it. Amounts are stored as integer billionths of a dollar rather than through `INCRBYFLOAT`, which accumulates rounding error over the tens of thousands of requests a busy day produces. Billionths rather than millionths because the residue does not average out: an agent repeats requests of near-identical shape, so the same fraction rounds the same way every time.

Each window is applied as one Lua script, in one round trip, so no other client sees a window holding part of a response and the exact-integer ceiling is checked before anything is written. That guarantee is per window, not across them: a response counting against a day budget and a month budget is two scripts, so a failure between the two leaves the day counted and the month not. Widening it needs a store operation that takes every window at once, tracked in [#536](https://github.com/pydantic/pydantic-ai-harness/issues/536).

A failure *after* the server has run a script does not say whether it committed — the connection can drop once `EVAL` has landed — so an `add` that errors leaves the outcome unknown rather than untried. Nothing retries it: counting a billed response twice is a direction the brake survives, and counting it zero times is not.

The cost of doing it server-side is a ceiling — the counters pass through Lua, whose numbers stop being exact integers above `2**53` billionths, about **$9,007,199** against a single key. Past that the store raises rather than rounding silently. Settling a protocol without that ceiling is [#532](https://github.com/pydantic/pydantic-ai-harness/issues/532).

The default store is built per capability, so two `SpendLimits` instances do not quietly share one counter. Pass the same store object to both when you want them to.

A store that fails does not fail quietly. An error reading the counter refuses the request, which is the safe direction. An error writing it propagates out of the run after the model has already answered and been charged. That is deliberate: a swallowed write would drift the counter down and weaken the gate, which is worse than a visible failure. If your deployment would rather keep the answer than the count, wrap the store and decide there.

Any object with `get` and `add` works, so a Postgres or DynamoDB counter is a small class rather than a fork.

Prices come from [genai-prices](https://github.com/pydantic/genai-prices) via `ModelResponse.cost()`, per response: cache and tier pricing are per request, so summing usage across requests and pricing the total gives the wrong number.

A model the registry does not know — a local deployment, a negotiated rate — is handled by `price`:

```
from decimal import Decimal
from pydantic_ai_harness import SpendLimits
SpendLimits(price=lambda response: Decimal('0.002') if response.model_name == 'internal-7b' else None)
```
An amount returned by `price` must be finite and not negative. Anything else — a credit, a `NaN`, an infinity — fails the run with `UserError`, because a credit moves a budget away from its ceiling and the other two are a broken pricing function rather than a price. The response is still recorded first: it was billed by the provider whatever the function returned, so its tokens and request count are accrued and `on_spend` fires before the error is raised.

Returning `None` falls through to the registry. When nothing can price a response, `on_unpriced` decides: `'zero'` (the default) counts it as free and increments `Spent.unpriced_requests` so the gap is visible, and `'raise'` fails the run with `UnpricedModelError`. Either way the response is recorded first and the tokens are counted, so a token ceiling still holds for a model with no price and an application that catches the error does not carry on against an understated counter. Under `'zero'` a USD ceiling is the one that cannot hold: nothing priceable accrues, so no number of such requests reaches it. That combination — `'zero'` plus a `usd` budget — warns once per model with `UnpricedModelWarning`, rather than once per request. If callers choose the model, prefer `'raise'` or supply `price`.

State lives across runs deliberately, so `for_run` is not overridden: a daily budget that reset every run would not be a daily budget. Per-run isolation comes from `Budget(window='run')`, whose key carries the run id.

`defer_loading=True` is refused. A deferred capability’s hooks do not run until the model loads it, so an exhausted budget would not stop a request and the requests made meanwhile would go uncounted — a brake the thing being braked decides when to apply.

The accrual happens in `wrap_model_request`, immediately around the provider call, and the capability declares itself innermost so that wrapper is the innermost one. Every other capability’s wrapper, and every capability’s `after_model_request`, therefore runs outside it and cannot reject a response the counter has not already seen.

`after_model_request` is the wrong hook for this. It runs once the whole wrap chain has returned, so a capability whose own `wrap_model_request` awaits the response and then raises `ModelRetry` sends the run straight to a fresh request and the rejected one — generated, billed, kept in history — is never counted. Ordering cannot reach that case: the rejecting capability need not be innermost, and one listed *before* `SpendLimits` still wraps outside it.

Wrapping also means a request the provider never saw is not charged for. `SkipModelRequest` from an earlier capability’s `before_model_request` reaches `after_model_request` with a response the run never paid for, but never reaches the wrapped handler.

What is left is siblings. Pydantic AI orders innermost capabilities against non-innermost ones only, and among themselves the one listed *later* nests further in. `TemporalDurability` and `InputGuardrail` also declare themselves innermost, so either of them listed after `SpendLimits` wraps inside it and can still reject a billed response before it is counted. List `SpendLimits` last among your innermost capabilities when that matters. Closing it outright needs a way to order innermost capabilities against each other, or the public composition-validation hook being decided in [pydantic-ai#5477](https://github.com/pydantic/pydantic-ai/issues/5477); tracked in [#534](https://github.com/pydantic/pydantic-ai-harness/issues/534).

**Durable execution.** `SpendLimits` is not supported inside a Temporal workflow.

The capability hooks run in workflow code; only the model request itself is the activity. Temporal replays workflow code, so it replays the accrual with it, and a window ends up counting the same response more than once — one `$1` model activity leaves `$2` in the store once the workflow replays. The day and month buckets have the same problem from the other side: they come from a wall clock the workflow sandbox restricts, and under time-skipping the workflow’s day and the key’s day drift apart.

The sandbox refusing that clock is what surfaces this first, and `SpendLimits` translates the error into what it means rather than into the setting that silences it. Passing the package through the sandbox removes the message, not the replay.

Refuse the workflow **admission** before starting it instead. That is why `exhausted()` works without a `RunContext`:

```
async def start_if_funded(limits: SpendLimits[None], tenant_id: str) -> None:
    if await limits.exhausted(scope=tenant_id):
        raise RuntimeError('daily budget exhausted')
    await workflow_handle.execute(...)
```
`exhausted` rather than `any(s.exhausted for s in await limits.status(...))`: `status` omits
the budgets it cannot resolve, and `any()` over what is left is a brake that passes having
inspected nothing — which is exactly what a `SpendLimits` whose budgets are all scoped returns when
the scope is missing. `exhausted` raises there instead, naming the budgets that need a
`scope` or a run context. Use `status` for a reading, `exhausted` for a decision.

Making the accrual replay-safe needs the store write to happen inside an activity, which the
capability cannot arrange without depending on `temporalio` and detecting the engine — so it
belongs in Pydantic AI core rather than here. Tracked in
[#531](https://github.com/pydantic/pydantic-ai-harness/issues/531).

For a per-run token ceiling and nothing else, Pydantic AI’s own
[`UsageLimits(total_tokens_limit=...)`](https://pydantic.dev/docs/ai/core-concepts/agent/#usage-limits)
does the same job in-process with no store and no capability. Reach for `Budget(tokens=..., window='run')` when the same configuration also has to express money, a longer window, a
tenant scope, or a counter shared between processes.

A refusal emits a `spend budget exhausted` span with `spend.budget` and `spend.window`. Accrual emits nothing: a span per model request would double the size of a trace without adding a decision. `spend.scope` is attached only when `RunContext.trace_include_content` is set, since a scope key is usually a tenant or user id and a trace has a wider audience than the application that produced it.

`Agent.from_spec` supports the part of the configuration a spec can express:

```
- SpendLimits:
    budgets:
      - {usd: '100', window: day}
      - {usd: '2000', window: month, warn_at: 0.8}
    on_unpriced: raise
```
`store`, `price`, `on_spend`, `clock`, and a budget’s `scope` take callables or live objects. A spec naming them is rejected rather than silently ignored, because a spec that promises per-tenant scoping and does not deliver it is worse than one that refuses to load.

The fields above are what `SpendLimits.from_spec` names in its signature, which is also what Pydantic AI reads to generate the spec’s JSON schema — so an editor following the `$schema` line completes and validates them. `BudgetSpec` is the entry shape, exported for anyone building a spec in code.

Source: [`pydantic_ai_harness/spend/`](https://github.com/pydantic/pydantic-ai-harness/tree/main/pydantic_ai_harness/spend/).

**Bases:** `AbstractCapability[AgentDepsT]`

Accumulate spend per window and refuse a request once a window is exhausted.

```
from decimal import Decimal
from pydantic_ai import Agent
from pydantic_ai_harness.spend import Budget, SpendLimits
agent = Agent(
    'openai:gpt-5.4',
    capabilities=[SpendLimits(budgets=[Budget(usd=Decimal('100'), window='day')])],
)
```
With no budgets the capability only reports, through `on_spend`. Add a
`Budget` with no ceiling to keep a running total that never blocks.

What the gate guarantees: no request **starts** after a budget is
exhausted. What it does not: that spend stays under the ceiling. The
request that crosses the line completes, and concurrent runs can each pass
the check before any of them records anything. This is a brake on a runaway
loop, not an accounting ledger.

State lives across runs on purpose, so `for_run` is left alone: a daily
budget that reset every run would not be a daily budget. Per-run isolation
comes from `Budget(window='run')`, whose key carries the run id.

Durable execution: not supported inside a Temporal workflow. The hooks run in
workflow code while only the model request is the activity, so Temporal replays
the accrual and a window counts the same response more than once. `exhausted()`
works without a `RunContext` so a workflow can at least be refused admission on
what is already recorded — but a workflow admitted that way records nothing of
its own, so it is a gate on the door, not a budget on what happens inside.
Tracked in [https://github.com/pydantic/pydantic-ai-harness/issues/531](https://github.com/pydantic/pydantic-ai-harness/issues/531).

Windows to accumulate against, and which of them can refuse a request.

**Type:** [`Sequence`](https://docs.python.org/3/library/typing.html#typing.Sequence)[`Budget`[`AgentDepsT`]] **Default:** `()`

Supplies the time that day and month windows are derived from.

It does not reach a default-constructed `store`, which keeps its own `utc_now` for
expiry. Both remain absolute instants, so a custom clock buckets on one and expires on
the other; pass the same callable to the store when that matters.

**Type:** [`Callable`](https://docs.python.org/3/library/typing.html#typing.Callable)[[], [`datetime`](https://docs.python.org/3/library/datetime.html#module-datetime)] **Default:** `utc_now`

Offer the agent a `get_spend` tool.

Off by default: a tool costs schema tokens on every request, and most applications want the number on a screen rather than in the model’s context.

**Type:** `bool`**Default:** `False`

Called with a `SpendSnapshot` after each response. May be sync or async.

**Type:** `SpendCallback` | `None`**Default:** `None`

What to do when a response cannot be priced.

`'zero'` counts it as free and increments `Spent.unpriced_requests`, so the
gap shows up instead of disappearing. `'raise'` fails the run with
`UnpricedModelError`. Tokens are counted either way, so a token ceiling
still holds for a model the registry does not know.

**Type:** [`Literal`](https://docs.python.org/3/library/typing.html#typing.Literal)[‘zero’, ‘raise’] **Default:** `'zero'`

Prices a response before the registry is consulted.

Returning `None` falls through to `genai-prices`. This is the way to charge
a self-hosted model, or a negotiated rate the public registry does not know.

An amount must be finite and not negative. Anything else fails the run with
`UserError`, after the response’s tokens and request count have been recorded:
a credit would move a budget away from its ceiling, and a NaN or an infinity
is a broken pricing function rather than a price.

**Type:** `PriceFunc` | `None`**Default:** `None`

Where counters live. The default holds them for the lifetime of the process.

**Type:** `SpendStore` **Default:** `field(default_factory=InMemorySpendStore)`

```
def __post_init__() -> None
```
Reject an `on_unpriced` that arrived as plain data and is not one of the two policies.

Anything other than `'raise'` behaves as `'zero'`, so a typo in a spec
would quietly turn unpriced responses free instead of failing the run.

`@async`

```
def before_model_request(
    ctx: RunContext[AgentDepsT],
    request_context: ModelRequestContext,
) -> ModelRequestContext
```
Refuse the request if any budget with a ceiling is already spent.

`@async`

```
def exhausted(
    ctx: RunContext[AgentDepsT] | None = None,
    *,
    scope: str | None = None,
) -> bool
```
Whether any budget this call can read is exhausted, refusing to guess about the rest.

An admission check, and only that. It reads the counters; it reserves nothing and records nothing, so work started on the strength of it goes unmeasured unless something else accrues it. Under Temporal that is the whole of what is available — the hooks cannot reach a shared store mid-run — so a workflow admitted here spends against a counter that never moves, and the next workflow is admitted on the same stale reading. Sound as a floor on runaway spend already recorded, not as a ceiling on what the workflow goes on to spend.

`status()` omits what it cannot resolve, and `any(...)` over the remainder is a brake
that silently checks nothing when every budget is scoped — so this raises instead,
naming the budgets that need a `scope` or a `ctx`.

`@classmethod`

```
def from_spec(
    cls,
    *,
    budgets: Sequence[BudgetSpec] = (),
    on_unpriced: Literal['zero', 'raise'] = 'zero',
    expose_tools: bool = False,
    id: str | None = None,
    description: str | None = None,
    defer_loading: bool = False,
    **unsupported: Any,
) -> SpendLimits[Any]
```
Build from an agent spec, covering the fields a spec can express.

Every parameter is named because that signature is what core reads to generate
the spec’s JSON schema: `build_schema_types` drops `*args`/`**kwargs`, so a
catch-all signature publishes the bare string `'SpendLimits'` and an editor
marks every documented `budgets:` block as invalid even though it loads.

`budgets` arrive as mappings and become `Budget` instances, with `usd`
accepted as a string so YAML cannot round a price through a float. The
callables and the store have no spec representation and are rejected
rather than dropped: a spec that promises per-tenant scoping and does
not deliver it is worse than a spec that refuses to load. `**unsupported`
stays so that rejection keeps naming the field; core drops it from the schema,
so it costs nothing there.

`SpendLimits`[[`Any`](https://docs.python.org/3/library/typing.html#typing.Any)]

```
def get_ordering() -> CapabilityOrdering
```
Sit innermost, so the accrual is the first thing to happen to a billed response.

Innermost puts this capability’s `wrap_model_request` closest to the provider call,
so every other capability’s wrapper — and every capability’s
`after_model_request` — runs outside the accrual and cannot reject a response the
counter has not already seen.

This orders against non-innermost capabilities only. Innermost members are not
ordered among themselves, and the one listed later nests further in, so another
innermost capability (`TemporalDurability`, `InputGuardrail`) placed after this one
still wraps inside it and can reject a billed response before it is counted. List
`SpendLimits` last among innermost capabilities where that matters; closing it
outright is [https://github.com/pydantic/pydantic-ai-harness/issues/534](https://github.com/pydantic/pydantic-ai-harness/issues/534).

`CapabilityOrdering`

`@classmethod`

```
def get_serialization_name(cls) -> str | None
```
Serialization name for agent-spec support.

```
def get_toolset() -> AgentToolset[AgentDepsT] | None
```
Offer `get_spend` when `expose_tools` is set.

[`AgentToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AgentToolset)[`AgentDepsT`] | `None`

`@async`

```
def status(
    ctx: RunContext[AgentDepsT] | None = None,
    *,
    scope: str | None = None,
) -> tuple[BudgetStatus, ...]
```
Where each budget stands.

Inside a run, pass `ctx` and every budget resolves. Without one — the
reading a cost display wants, and the check to make before starting a
durable workflow whose hooks cannot reach a shared store — budgets on a
`run` or `conversation` window are omitted, since those periods have no
meaning outside a run, and a budget declaring a `scope` is omitted
unless `scope` names the partition to read, since its callable has no
run context to resolve against.

Reach for [`exhausted`](/docs/ai/harness/spend/#pydantic_ai_harness.spend.SpendLimits.exhausted) when the
answer gates something: `any(s.exhausted for s in ...)` over a tuple that happens to
be empty is a brake that reads as enforcement and inspects nothing, and a `SpendLimits`
whose budgets are all scoped returns exactly that tuple.

[`tuple`](https://docs.python.org/3/library/stdtypes.html#tuple)[`BudgetStatus`, …]

`@async`

```
def wrap_model_request(
    ctx: RunContext[AgentDepsT],
    *,
    request_context: ModelRequestContext,
    handler: WrapModelRequestHandler,
) -> ModelResponse
```
Price what the provider returned and add it to every window, before anything can reject it.

The accrual belongs here rather than in `after_model_request` because
`after_model_request` runs outside this chain, once the whole chain has returned.
A capability whose own `wrap_model_request` awaits the response and then raises
`ModelRetry` sends the run straight to a fresh request, and the response it
rejected — generated, billed, and kept in history — is never counted. Ordering
cannot close that: the rejecting wrapper does not have to be innermost, and one
listed *before* this capability still nests outside it.

Wrapping is also why a request the provider never saw is not charged for.
`SkipModelRequest` from an earlier `before_model_request` reaches
`after_model_request` with a response the run never paid for; it does not reach
`handler`, so nothing accrues here.

**Bases:** `Generic[AgentDepsT]`

One spend window: what it limits, over what period, for whom.

A budget with neither `usd` nor `tokens` is a pure counter: it accumulates
and reports, and never stops a run. That is how per-tenant accounting with
no cap is expressed.

Generic in the agent’s dependency type so `scope` is checked against it: the
parameter comes from the `Agent` the capability is passed to, so a scope
reaching for a field the deps do not have is a type error rather than an
`AttributeError` on the first request.

```
from decimal import Decimal
from pydantic_ai_harness.spend import Budget
Budget(usd=Decimal('100'), window='day')
Budget(usd=Decimal('10'), window='day', scope=lambda ctx: ctx.deps.tenant_id)
Budget(window='month', name='accounting')  # counts, never blocks
```
Whether this budget can refuse a request, rather than only counting.

**Type:** `bool`

Distinguishes budgets sharing a window and scope. Part of the store key.

**Type:** `str`**Default:** `'default'`

How long a store may keep this window’s counter after its last write.

`'window default'` takes the horizon from `window` (see `_TTLS`). A time window
may expire freely once it has rolled over, but `run` and `conversation` buckets
never roll over, so their defaults are a compromise between never expiring and
growing the store without bound — and a conversation resumed past the horizon
starts again from zero. Set `'forever'` where that matters and the keys are
cleaned up some other way, or a `timedelta` to pick the horizon outright.

**Type:** `timedelta` | [`Literal`](https://docs.python.org/3/library/typing.html#typing.Literal)[‘window default’, ‘forever’] **Default:** `'window default'`

Partitions the counter — per tenant, per user, per agent. `None` counts globally.

**Type:** [`Callable`](https://docs.python.org/3/library/typing.html#typing.Callable)[[[`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)[`AgentDepsT`]], [`str`](https://docs.python.org/3/library/stdtypes.html#str)] | `None`**Default:** `None`

Ceiling in total tokens. `None` means this budget does not limit tokens.

**Type:** [`int`](https://docs.python.org/3/library/functions.html#int) | `None`**Default:** `None`

How long a store may keep this window’s counter after its last write.

**Type:** `timedelta` | `None`

Ceiling in US dollars. `None` means this budget does not limit spend.

**Type:** `Decimal` | `None`**Default:** `None`

Fraction of the ceiling past which `BudgetStatus.warning` is set. Never blocks.

**Type:** [`float`](https://docs.python.org/3/library/functions.html#float) | `None`**Default:** `None`

The period the ceiling applies to.

**Type:** `Window` **Default:** `'day'`

```
def __post_init__() -> None
```
Reject configurations that would quietly misbehave rather than fail.

A ceiling of zero or less makes a budget exhausted before anything is
spent, so the first request is refused with no way to tell that from a
real overspend — and `usd: 0` in a spec is far more likely to mean “no
limit”, which is what `None` says. A `warn_at` on a budget with no ceiling has nothing to be a
fraction of, so it can never fire. Both read as configuration and behave
as breakage, which is why they are errors here rather than surprises
later.

What one model response cost, and where every budget stands after it.

One entry per configured budget, in the order they were declared.

**Type:** [`tuple`](https://docs.python.org/3/library/stdtypes.html#tuple)[`BudgetStatus`, …]

The model that produced the response, or `None` if it reported none.

Whether `usd` is a real price or a stand-in zero.

**Type:** `bool`

The response’s usage verbatim, including cache reads, writes, and audio.

**Type:** `RequestUsage`

What this response cost. Zero when it could not be priced.

**Type:** `Decimal`

A budget and how much of it is left.

The budget this describes.

Unparameterised because this is a reading: the dependency type only types
`Budget.scope`, and nothing calls a scope through a status.

**Type:** `Budget`[[`Any`](https://docs.python.org/3/library/typing.html#typing.Any)]

Whether a further request would be refused.

**Type:** `bool`

The store key it accumulates under. Useful for debugging a scope or window.

**Type:** `str`

`None` when the budget sets no token limit.

`None` when the budget sets no USD limit.

**Type:** `Decimal` | `None`

What the budget’s current window has accumulated.

**Type:** `Spent`

Whether spend has crossed `Budget.warn_at`. Always `False` without one.

**Type:** `bool`

Everything one window has accumulated so far.

Model requests recorded against this window.

**Type:** `int`**Default:** `0`

Total tokens, counted whether or not the request could be priced.

**Type:** `int`**Default:** `0`

How many of `requests` had no resolvable price, so `usd` understates them.

**Type:** `int`**Default:** `0`

Priced cost. Requests with no resolvable price contribute nothing here.

**Type:** `Decimal` **Default:** `Decimal(0)`

**Bases:** `Protocol`

Reads and accumulates the counter behind each budget window.

`add` returns the state **after** the increment so an atomic backend can
answer without a second round trip. `requests` is an explicit parameter
rather than an implied `+= 1` so a reconciler correcting drift against an
external source can post a delta (including a negative `usd`) without
inflating the request count, and without the protocol growing a second
method for it.

`@async`

```
def add(
    key: str,
    *,
    usd: Decimal,
    tokens: int,
    requests: int,
    unpriced: int,
    ttl: timedelta | None,
) -> Spent
```
Add to `key` and return the result. `ttl` is how long the key may be kept.

`Spent`

`@async`

```
def get(key: str) -> Spent
```
What `key` has accumulated. A key that was never written reads as zero.

`Spent`

Counters for the lifetime of one process.

Catches a runaway loop inside the worker it runs in. It does not enforce a
budget across processes: every worker of a queue would keep its own count,
which is what a shared store such as
[`RedisSpendStore`](/docs/ai/harness/spend/#pydantic_ai_harness.spend.RedisSpendStore) is for.

Supplies the time expiry is measured against.

**Type:** [`Callable`](https://docs.python.org/3/library/typing.html#typing.Callable)[[], [`datetime`](https://docs.python.org/3/library/datetime.html#module-datetime)] **Default:** `utc_now`

Writes between expiry sweeps.

Expiry cannot wait for the next read of a key: a day window produces a new key each day, so yesterday’s is never asked for again. The scan is linear in resident keys, so it is amortised over this many writes rather than run on each one, which bounds dead entries to roughly that many. Lower it where scopes are high-cardinality and memory matters more than the scan.

**Type:** `int`**Default:** `256`

```
def __len__() -> int
```
How many windows are still live.

Rolled-over entries are excluded whether or not the amortised sweep has reached them
yet, so this counts what is being tracked rather than what happens to be resident.
That is one entry per budget, scope and period — and for a `run` or `conversation`
budget the period is an id, so the count grows with traffic until those entries reach
their horizon. Worth watching there and wherever scopes are high-cardinality.

Defining `__len__` makes an empty store falsy, so write `if store is not None`.

`@async`

```
def add(
    key: str,
    *,
    usd: Decimal,
    tokens: int,
    requests: int,
    unpriced: int,
    ttl: timedelta | None,
) -> Spent
```
Add to `key` and return the result.

The mutation spans no `await`, so concurrent runs on one event loop
cannot interleave halfway through it. The lock covers the case that is
not free: `run_sync` called from a thread pool, or a free-threaded
interpreter, where a read-modify-write loses updates in the direction
that under-counts spend.

`Spent`

`@async`

```
def get(key: str) -> Spent
```
What `key` has accumulated, treating an expired key as absent.

Under the lock, because `_live` deletes the key it finds expired: unlocked, that
`del` races the `_sweep` iteration inside `add` (`RuntimeError: dictionary changed size during iteration`) and a second concurrent reader (`KeyError`). Reachable
whenever the guard is shared across threads — `run_sync` from a pool, a sync
endpoint — and any key is read past its horizon.

`Spent`

Spend counters in Redis, so every worker enforces one budget.

One hash per window, holding the four counters as integers.

```
from redis.asyncio import Redis
from pydantic_ai_harness.spend import RedisSpendStore
store = RedisSpendStore(Redis.from_url('redis://localhost'))
```
A read and the increment that follows it are separate round trips, so concurrent runs can each observe a budget as unexhausted and push past it together. That is the same overshoot the in-process store has, widened by the number of workers; see the README on what the gate does and does not guarantee.

Any client exposing `hgetall` and `eval`.

**Type:** `RedisClient`

Namespace for the keys, so a shared Redis stays tidy.

**Type:** `str`**Default:** `'pydantic-ai-harness:spend'`

`@async`

```
def add(
    key: str,
    *,
    usd: Decimal,
    tokens: int,
    requests: int,
    unpriced: int,
    ttl: timedelta | None,
) -> Spent
```
Add to `key` and return the result.

One round trip, and one window. The script returns each new total, so the result needs no second read.

A failure before the server runs the script — the client cannot connect, the
request never lands — writes nothing. A failure after it does not say which:
the connection can drop once `EVAL` has already committed, so an error here
means the outcome is unknown rather than that nothing happened. Retrying an
`add` therefore risks counting the response twice, which is why
`SpendLimits.wrap_model_request` does not retry and lets the error end the run
instead. Over-counting a response the provider did bill is the direction a
brake can survive; under-counting is not.

One window, because the key is one window. A response counting against a day and
a month budget is two calls, so a failure between them leaves the day counted and
the month not; see `SpendLimits.wrap_model_request` and
[https://github.com/pydantic/pydantic-ai-harness/issues/536](https://github.com/pydantic/pydantic-ai-harness/issues/536).

`Spent`

`@async`

```
def get(key: str) -> Spent
```
What `key` has accumulated. An absent hash reads as zero.

`Spent`

**Bases:** `UsageLimitExceeded`

Raised when a [`Budget`](/docs/ai/harness/spend/#pydantic_ai_harness.spend.Budget) is exhausted.

Subclasses [`UsageLimitExceeded`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UsageLimitExceeded)
so an application that already stops on a usage limit stops on a spend limit
too, while code that needs to tell “the daily budget is gone” from “this run
used too many tokens” can catch this type specifically.

**Bases:** `UserError`

Raised when `on_unpriced='raise'` and no price could be resolved for a response.

Either the model is absent from the `genai-prices` registry (a local or
custom deployment) or the response carries no model name. Supply
`SpendLimits.price` to price it yourself, or use `on_unpriced='zero'` to
count the request as free and surface it as `Spent.unpriced_requests`.

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/spend
