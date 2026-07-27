---
type: Web Page
title: Managed Prompt | Pydantic Docs
description: Back a Pydantic AI agent's instructions with a Logfire-managed prompt
  so you can version, label, and roll it out without redeploying.
resource: https://pydantic.dev/docs/ai/harness/managed-prompt
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Managed Prompt

`ManagedPrompt` backs an agent’s instructions with a
[Logfire-managed prompt](https://logfire.pydantic.dev/docs/reference/advanced/prompt-management/),
so you can iterate on your system prompt from the Logfire UI — versioned, labelled, and rolled
out — without touching code or redeploying. It’s a Pydantic AI [capability](/docs/ai/harness/), so you
wire it in through the `capabilities=` parameter on `Agent`.

Install the `logfire` extra:

Prompts are critical to agent behavior, but iterating on them through the normal edit -> review -> deploy loop is slow. You can’t easily A/B test a change, and you can’t roll it back the moment it misbehaves in production without shipping a new build.

`ManagedPrompt` moves the prompt out of your codebase and into Logfire’s managed-variable store.
It declares the backing managed variable for you and resolves it **once per run**, feeding the
resolved value into the agent’s instructions. Resolution happens inside the run’s
[ wrap_run](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_run)
hook, using the

[as a context manager that stays open for the whole run — so the selected label and version are attached as baggage to every child span of the agent run. You get a direct correlation between a run’s behavior and the exact prompt version that produced it, plus instant iteration and rollback from the Logfire UI.](https://logfire.pydantic.dev/docs/reference/advanced/managed-variables/)

`ResolvedVariable`Pass the prompt name and a default value. The name `support_agent` is declared as the managed
variable `prompt__support_agent` — the naming Logfire’s Prompt management uses (hyphens in a name
become underscores). The `default` keeps the agent working until a remote value is published, so
your code always runs even before you create the prompt in Logfire.

```
import logfire
from pydantic_ai import Agent
from pydantic_ai_harness.logfire import ManagedPrompt
logfire.configure()
agent = Agent(
    'openai:gpt-5',
    capabilities=[
        ManagedPrompt(
            'support_agent',
            default='You are a helpful customer support agent. Be friendly and concise.',
            label='production',
        )
    ],
)
result = agent.run_sync('My order never arrived.')
print(result.output)
```
Pinning `label='production'` is the recommended default: the resolved value only changes on a
deliberate prompt rollout, which keeps the provider prompt cache hot (see
[Prompt-cache trade-off](#prompt-cache-trade-off) below).

For deterministic A/B assignment (the same user always sees the same label), pass a
`targeting_key`. It can be a static string or a callable that derives the key from the
[ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) — handy when the
key lives in your agent’s 

`deps`:```
from dataclasses import dataclass
from pydantic_ai import Agent
from pydantic_ai_harness.logfire import ManagedPrompt
@dataclass
class Deps:
    user_id: str
agent = Agent(
    'openai:gpt-5',
    deps_type=Deps,
    capabilities=[
        ManagedPrompt(
            'support_agent',
            default='You are a helpful customer support agent.',
            targeting_key=lambda ctx: ctx.deps.user_id,
        ),
    ],
)
```
Pass `attributes` (a mapping, or a callable returning one) for condition-based targeting rules.
When `label` is omitted, the variable’s rollout and targeting rules pick the label. When both
`targeting_key` and `attributes` are omitted, Logfire falls back to its own targeting context and
then to the active trace id.

For Logfire-side targeting that lives outside the agent (e.g. set once per request handler), use
Logfire’s
[ targeting_context](https://logfire.pydantic.dev/docs/reference/advanced/managed-variables/) in
an outer scope; 

`ManagedPrompt` only needs `targeting_key` / `attributes` when the key comes from
the agent’s `RunContext`.By default the resolved prompt is used verbatim. Pass `render_template=True` to render it as a
Handlebars template against the agent’s `deps` — the same mechanism as
[ TemplateStr](/docs/ai/api/pydantic-ai/agent/) — so 

`{{field}}` is filled
from `deps`:```
from dataclasses import dataclass
from pydantic_ai import Agent
from pydantic_ai_harness.logfire import ManagedPrompt
@dataclass
class Deps:
    customer_name: str
agent = Agent(
    'openai:gpt-5',
    deps_type=Deps,
    capabilities=[
        ManagedPrompt(
            'support_agent',
            default='You are helping {{customer_name}}. Be friendly and concise.',
            render_template=True,
        ),
    ],
)
```
Rendering requires `pydantic-handlebars` (install `pydantic-ai-slim[spec]`). It is off by default.

The resolved value lands in the agent’s **system instructions**. Provider prompt caches (Anthropic,
OpenAI, etc.) key strictly by prefix — `tools -> system -> messages` — so any change to the system
block invalidates the cached prefix for the affected runs.

| Mode | Cache impact | 
|---|---|
| Pinned `label='production'`, no rollout split | Cache-stable.The value only changes on a deliberate prompt rollout, which is the same cost as a redeploy. | 
| Percentage rollout across labels (no `label=`) | Different runs land on different labels -> splits the cache into one lane per label. | 
| `targeting_key`per user/tenant with multiple labels in play | Cache lanes per assigned label; deterministic per key but still N lanes overall. | 
| Mid-traffic label flip in the Logfire UI | One-shot cold-invalidation for everyone on that label. | 

In short: pinning a `label` keeps the cache hot; using `ManagedPrompt` as an A/B platform is opt-in
cache cost. If you don’t need rollouts, `label='production'` is the recommended default.

Declaring the same name more than once is fine — each `ManagedPrompt` builds its own backing
variable, so sharing a prompt across several agents just works. Pass an existing
[ logfire.variables.Variable](https://logfire.pydantic.dev/docs/reference/advanced/managed-variables/)
as the first argument instead of a name when you want to declare the variable yourself — for
example a template variable, or one registered for 

`variables_push`:```
import logfire
from pydantic_ai import Agent
from pydantic_ai_harness.logfire import ManagedPrompt
logfire.configure()
support_prompt = logfire.var(
    name='prompt__support_agent',
    type=str,
    default='You are a helpful customer support agent. Be friendly and concise.',
)
agent = Agent('openai:gpt-5', capabilities=[ManagedPrompt(support_prompt, label='production')])
```
When `name` is a prompt name (not a `Variable`), pass `logfire_instance=` to declare the variable
on a specific Logfire instance instead of the module-level default. `default` is required when
`name` is a prompt name and is ignored when you pass a `Variable` (which already carries its own
default and instance).

- **Resolves once per run.**A label flip or rollout change that lands in Logfire mid-run is not picked up until the next run starts — the trade-off for run-stable instructions and a single baggage scope across all child spans.
- **Runs outermost.**The capability wraps- `Instrumentation`
- **Concurrency-safe.**Resolution is isolated per run via a context variable, so a single capability instance is safe to share across concurrent runs.
- **Inspectable mid-run.**- `ManagedPrompt.resolved`exposes the active run’s- `ResolvedVariable`(- `value`,- `label`,- `version`,- `reason`) for inspection — e.g. from inside a tool. It is- `None`outside a run.

The resolved prompt is a `str`. Pass the bare prompt name (the `prompt__` prefix and
hyphen-to-underscore normalization are applied for you) and a `default`, then use `label`,
`targeting_key`, `attributes`, `render_template`, and `logfire_instance` to control resolution.

**Bases:** `AbstractCapability[AgentDepsT]`

Back an agent’s instructions with a Logfire-managed prompt.

**Prompt-cache trade-off:** the resolved value lands in the system instructions block, so any
Logfire-side change to the prompt (new version rollout, label flip, A/B targeting) invalidates
the provider’s prompt cache for the affected runs. Pin a `label` (e.g. `'production'`) for the
cache-stable path; treat percentage rollouts and per-user targeting as opt-in cache cost. See
the README’s “Prompt-cache trade-off” section for the full picture.

Pass the managed prompt name and a default value and the capability declares the backing
[managed variable](https://logfire.pydantic.dev/docs/reference/advanced/managed-variables/)
for you — a name of `support_agent` resolves the variable `prompt__support_agent`, matching
the naming Logfire’s [Prompt management](https://logfire.pydantic.dev/docs/reference/advanced/prompt-management/)
uses. You can iterate on the prompt from the Logfire UI — versioned, labelled, and rolled
out — without redeploying, while the code default keeps the agent working when no remote
value is available.

```
import logfire
from pydantic_ai import Agent
from pydantic_ai_harness.logfire import ManagedPrompt
logfire.configure()
agent = Agent(
    'openai:gpt-5',
    capabilities=[
        ManagedPrompt(
            'support_agent',
            default='You are a helpful customer support agent. Be friendly and concise.',
            label='production',
        )
    ],
)
result = agent.run_sync('My order never arrived.')
```
The prompt value is resolved **once per run**, inside the run’s
[ wrap_run](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.wrap_run) hook, using the

`ResolvedVariable` as a context manager that stays open for the
whole run — so the selected label and version are attached as baggage to every child span
of the agent run.Declaring the same name more than once is fine — each `ManagedPrompt` constructs its own
backing variable, so sharing a prompt across several agents just works. Pass an existing
`logfire.variables.Variable` as `name` instead of a prompt name
when you want to use a variable you defined yourself (for example a `template_var`, or one
registered for [ variables_push](https://logfire.pydantic.dev/docs/api/logfire/#logfire.Logfire.variables_push)).

The managed prompt name (declared as the variable `prompt__<name>`), or a pre-built `logfire.Variable`.

Code-default prompt text. Required when `name` is a prompt name; ignored when `name` is a `Variable`.

**Type:** [ str](https://docs.python.org/3/library/stdtypes.html#str) | 

`None`**Default:**

`None`Explicit targeting label on the Logfire managed prompt to resolve (e.g. `'production'`).
When `None`, the targeting rules on the managed variable select the label.

**Type:** [ str](https://docs.python.org/3/library/stdtypes.html#str) | 

`None`**Default:**

`None`Stable key that seeds Logfire’s deterministic rollout assignment — the same key always
lands in the same percentage bucket, so a given user keeps the same label across runs.
Accepts a static value or a callable that derives it from the
[ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext). When 

`None`, Logfire falls back to its own
targeting context and then the active trace id.**Type:** [ str](https://docs.python.org/3/library/stdtypes.html#str) | 

[[[](https://docs.python.org/3/library/typing.html#typing.Callable)

`Callable`[[](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext``AgentDepsT`]], [|](https://docs.python.org/3/library/stdtypes.html#str)

`str`[] |](https://docs.python.org/3/library/constants.html#None)

`None`

`None`**Default:**

`None`Attributes for condition-based targeting rules, or a callable that derives them
from the [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext).

**Type:** [ Mapping](https://docs.python.org/3/library/typing.html#typing.Mapping)[

[,](https://docs.python.org/3/library/stdtypes.html#str)

`str`[] |](https://docs.python.org/3/library/typing.html#typing.Any)

`Any`[[[](https://docs.python.org/3/library/typing.html#typing.Callable)

`Callable`[[](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext``AgentDepsT`]], [[](https://docs.python.org/3/library/typing.html#typing.Mapping)

`Mapping`[,](https://docs.python.org/3/library/stdtypes.html#str)

`str`[] |](https://docs.python.org/3/library/typing.html#typing.Any)

`Any`[] |](https://docs.python.org/3/library/constants.html#None)

`None`

`None`**Default:**

`None`When `True`, render the resolved prompt as a Handlebars template against the agent’s
`deps` (the same mechanism as [ TemplateStr](/docs/ai/api/pydantic-ai/template/#pydantic_ai.template.TemplateStr)); 

`{{field}}` is
filled from `deps`. Requires `pydantic-handlebars` (install `pydantic-ai-slim[spec]`).
Defaults to `False`, so the resolved prompt is used verbatim.**Type:** `bool`**Default:** `False`

Logfire instance to resolve the variable on. When `None`, the global default instance
(the one backing the module-level `logfire.var`) is used. Ignored when
`name` is a `Variable`.

**Type:** `Logfire` | `None`**Default:** `None`

The prompt resolution for the active run, or `None` outside a run.

Exposes the full `ResolvedVariable` (`value`, `label`,
`version`, `reason`, …) so callers can inspect which prompt version is in play.

**Type:** `ResolvedVariable`[[ str](https://docs.python.org/3/library/stdtypes.html#str)] | 

`None````
def get_ordering() -> CapabilityOrdering
```
Run outermost so the prompt’s baggage envelops the whole run, including the run span.

`CapabilityOrdering`

```
def get_instructions() -> Callable[[RunContext[AgentDepsT]], str | None]
```
Provide the resolved prompt to the agent’s system prompt.

[ Callable](https://docs.python.org/3/library/typing.html#typing.Callable)[[

[[](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext``AgentDepsT`]], [|](https://docs.python.org/3/library/stdtypes.html#str)

`str`[]](https://docs.python.org/3/library/constants.html#None)

`None``@async`

```
def wrap_run(
    ctx: RunContext[AgentDepsT],
    *,
    handler: WrapRunHandler,
) -> AgentRunResult[Any]
```
Resolve the prompt once and keep its baggage active for the duration of the run.

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/managed-prompt
