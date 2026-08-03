---
type: Web Page
title: Subagents | Pydantic Docs
description: Let an agent delegate self-contained tasks to named child agents via
  a single delegate_task tool, with per-delegate budgets and failure handling.
resource: https://pydantic.dev/docs/ai/harness/subagents
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Subagents

`SubAgents` lets an agent delegate self-contained tasks to named child agents. It takes a sequence of `SubAgent` entries and exposes a single `delegate_task(agent_name, task)` tool. Each delegation runs the chosen sub-agent in its own run — with its own message history, so it never sees the parent conversation — and returns its output to the parent.

The API may change between releases. Where practical, breaking changes ship with a deprecation warning.

A single agent that does everything accumulates a large tool set and a long context. Splitting the work across specialized sub-agents keeps each context focused, but wiring up delegation by hand means writing a tool per agent, forwarding deps, threading usage limits, and telling the model what it can delegate to.

`SubAgents` takes a sequence of `SubAgent` entries and exposes a single `delegate_task(agent_name, task)` tool. Each delegation runs the chosen sub-agent in its own run — with its own message history, so it never sees the parent conversation — and returns its output to the parent. The available sub-agents are listed in the system prompt as a static instruction, so the listing stays in the cached prefix.

```
from pydantic_ai import Agent
from pydantic_ai_harness.subagents import SubAgent, SubAgents
researcher = Agent('anthropic:claude-sonnet-4-6', name='researcher', description='Researches a topic and reports findings')
writer = Agent('anthropic:claude-sonnet-4-6', name='writer', description='Turns notes into polished prose')
orchestrator = Agent(
    'anthropic:claude-opus-4-7',
    capabilities=[SubAgents(agents=[SubAgent(researcher), SubAgent(writer)])],
)
result = orchestrator.run_sync('Research the history of TLS and write a one-paragraph summary.')
print(result.output)
```
A delegate’s name — how the parent model refers to it, and how it is listed in the prompt — is the agent’s own `name`, or a `SubAgent(name=...)` override. Two delegates resolving to the same name is an error, and an agent with no name and no override is rejected.

| Tool | Purpose | 
|---|---|
| `delegate_task(agent_name, task)` | Run the named sub-agent on a self-contained task and return its output. | 

- The sub-agent runs with its own message history, so `task` must be self-contained.
- An unknown `agent_name` raises`ModelRetry` , so the model can correct itself.
- The result returned to the parent is `str(result.output)` .
- With a `models` menu configured, the tool takes an extra`model` argument (see below).

- **Deps are forwarded.** The parent run’s`deps` are passed to each sub-agent, so sub-agents share the parent’s`AgentDepsT` (enforced by the type signature — every sub-agent is an`AbstractAgent[AgentDepsT, Any]` ).
- **Usage is shared by default.** The parent’s`usage` is passed to each sub-agent run, so token usage aggregates and a parent`usage_limits` applies across the whole agent tree. Set`forward_usage=False` to give each sub-agent run its own accounting.
- **Tools can be inherited.** With`inherit_tools=True` , the parent agent’s own tools (registered directly or via`toolsets` ) are added to each sub-agent run, on top of the sub-agent’s own. Tools contributed by the parent’s capabilities are not inherited: they are bound to capability instances registered in the parent run, and would arrive without the hooks and instructions they depend on. Use`shared_capabilities` to give sub-agents a capability. This also excludes the delegate tool itself, so a sub-agent can’t recurse into further delegation. Off by default.
- **Capabilities can be shared.**`shared_capabilities` are applied to every sub-agent run — e.g. give all sub-agents a common guardrail, memory, or planning capability without rebuilding each`Agent` .
- **Sub-agent events can be streamed.** Pass an`event_stream_handler` and it’s forwarded to each sub-agent run, so the sub-agent’s model-streaming and tool events surface to the caller (the handler receives the sub-agent’s own`RunContext` ).

Each `SubAgent` carries its own budgets, so one delegate’s controls do not touch the others. A `SubAgent` with no controls set runs with the `SubAgents` defaults.

```
from pydantic_ai import Agent
from pydantic_ai.usage import UsageLimits
from pydantic_ai_harness.subagents import SubAgent, SubAgents
reproducer = Agent('anthropic:claude-sonnet-4-6', instructions='Reproduce the reported bug from a minimal script.')
librarian = Agent('anthropic:claude-sonnet-4-6', instructions='Find relevant docs, issues, and prior art.')
orchestrator = Agent(
    'anthropic:claude-opus-4-7',
    capabilities=[
        SubAgents(
            agents=[
                SubAgent(reproducer, usage_limits=UsageLimits(request_limit=35), timeout_seconds=600, max_calls=1),
                SubAgent(librarian, usage_limits=UsageLimits(request_limit=18), timeout_seconds=300, max_calls=2),
            ]
        )
    ],
)
```
| Field | Effect | 
|---|---|
| `models` | Which keys of the `SubAgents` model menu this delegate may run on, and which one it runs on by default: the first key listed. See “Per-delegation model selection” below. | 
| `usage_limits` | A request/token budget for one delegation. The child runs with its own usage accounting, so the budget counts only that child’s requests and tokens (not the parent’s or siblings’), even when `forward_usage=True` . The tradeoff: that child’s tokens no longer aggregate into the parent’s`usage` . Reaching the budget is a soft outcome (see below), not a run-stopping`UsageLimitExceeded` . | 
| `timeout_seconds` | A wall-clock budget for one delegation. When the child exceeds it, its run is cancelled and the parent gets a soft steering message instead of hanging on the child. The cancelled child’s `event_stream_handler` (if any) stops receiving events without a terminal event. | 
| `max_calls` | The maximum number of delegations to this sub-agent per parent run. Once reached, further delegations return a soft budget-exhausted message without running the child. Counts are scoped to one `Agent.run` (a`run_id` ) and cleared when it ends, so each parent run and each level of a nested tree budgets independently. | 
| `on_failure` | A steering message returned to the parent for any soft degradation of this delegate, in place of the built-in default. Setting it also makes child failures soft (see below). | 
| `contain_errors` | Whether an unexpected crash in this delegate is caught and returned to the parent as a bounded `ModelRetry` instead of aborting the parent run (see below). Unset inherits the`SubAgents(contain_errors=...)` default (off). | 

The orchestrator knows how hard a task is at the moment it writes the brief, so it is the right place to decide which model runs it. Configure a `models` menu and the delegate tool gains a `model` argument that names one of its keys.

```
from pydantic_ai import Agent
from pydantic_ai.settings import ModelSettings
from pydantic_ai_harness.subagents import ModelOption, SubAgent, SubAgents
reviewer = Agent('anthropic:claude-sonnet-4-6', name='reviewer', description='Reviews a diff')
linter = Agent('anthropic:claude-sonnet-4-6', name='linter', description='Runs the linter and reports failures')
orchestrator = Agent(
    'anthropic:claude-opus-4-7',
    capabilities=[
        SubAgents(
            agents=[SubAgent(reviewer), SubAgent(linter, models=['fast'])],
            models={
                'fast': 'anthropic:claude-haiku-4-5',
                'standard': 'anthropic:claude-sonnet-4-6',
                'deep': ModelOption(
                    'anthropic:claude-opus-4-7',
                    description='hard reasoning, multi-file changes',
                    settings=ModelSettings(thinking='xhigh'),
                ),
            },
        )
    ],
)
```
- **Off by default.** With no`models` menu the`model` argument is not in the tool schema at all, and every delegation runs exactly as it did before.
- **The keys are the interface.** They are listed in the system prompt with each entry’s model and description, and the tool schema offers them as an enum, so the model picks from the menu instead of inventing a model name. Name them for the job (`'fast'` ,`'deep'` ), not for the vendor.
- **An entry is a model or a `ModelOption`.**`ModelOption(model, description=..., settings=...)` adds a routing hint and per-option`ModelSettings` , so one key can mean “same model, more thinking”. Those settings merge over the sub-agent’s own`model_settings` , which keeps the parts the option does not set.
- **Resolution order.** The key the parent passed, else the delegate’s first allowed key (see below), else the delegate’s own model, else the parent run’s model.
- **A delegate can be restricted.**`SubAgent(linter, models=['fast'])` pins that delegate to`fast` : the first listed key is what it runs on when the parent passes no`model` , and the others are refused with a`ModelRetry` . The restriction is rendered in the prompt listing (`- linter: Runs the linter (models: fast)` ). Restricting to a key the menu does not define is a`ValueError` at construction.
- **A rejected key costs nothing.** An unknown or unavailable key comes back as a`ModelRetry` listing the valid options, before the delegate’s`max_calls` budget is charged.

A *soft outcome* returns a steering message to the parent as a normal tool result, so its model reads the message and decides what to do next (rather than immediately re-delegating, which a `ModelRetry` invites). A timeout, a reached `usage_limits` budget, and an exhausted `max_calls` budget are always soft. When `on_failure` is set, the message it carries replaces the built-in default for these outcomes.

A sub-agent run that fails with a *soft model error* (`ModelRetry`, `UnexpectedModelBehavior`, e.g. it exhausted its own retries) is, by default, converted into a `ModelRetry` for the parent — so the parent’s model sees `Sub-agent '<name>' failed: ...` and can react by re-delegating. The delegate tool defaults to `tool_retries=2`, so the parent aborts only after that many consecutive delegate failures; the counter resets after any successful delegation. Raise `tool_retries` to tolerate a flakier sub-agent, or set `None` to inherit the parent agent’s default tool retries. Set `on_failure` for a delegate to make its failures soft instead: the child error returns the `on_failure` message as a normal tool result.

Hard errors propagate to stop the whole run. A `UsageLimitExceeded` from a child that has *no* per-delegate `usage_limits` (so it shares the parent’s accounting) means the whole tree is out of budget and propagates; a child reaching its *own* `usage_limits` is soft, as above.

An *unexpected crash* — any other exception the child raises, such as a provider `ModelAPIError`/`FallbackExceptionGroup` or a plain `ValueError` from a bad tool argument — propagates by default and aborts the parent run. Set `contain_errors=True` (per delegate, or as the `SubAgents` default) to catch it and return it to the parent as a bounded `ModelRetry` instead, so one delegate crash cannot kill the whole run. Containment stays loud: the exception rides the retry message (`Sub-agent '<name>' crashed: ...`), it is logged via the standard `logging` module, and `tool_retries` still bounds consecutive crashes into an abort. This is orthogonal to `on_failure` — a contained crash always raises the loud retry, never the soft `on_failure` return, so a genuine bug is never masked as success. Cancellation, a shared `UsageLimitExceeded`, pydantic-ai control-flow signals (`CallDeferred`, `ApprovalRequired`, the `Skip*` signals), and `UserError` always propagate regardless of `contain_errors`.

The sub-agents are listed in the system prompt via `get_instructions`, using each agent’s `description` (or a `SubAgent(description=...)` override). A sub-agent with no description is listed by name alone.

A repo’s markdown agent definitions become delegates without writing any `Agent` code. By default every `*.md` file under the conventional folders is loaded as a sub-agent, alongside the explicitly-passed `agents`.

```
from pydantic_ai import Agent
from pydantic_ai_harness.subagents import SubAgents
orchestrator = Agent(
    'anthropic:claude-opus-4-7',
    capabilities=[SubAgents(inherit_tools=True)],  # auto-loads ./.agents/agents/ and ~/.agents/agents/
)
```
`agent_folders` controls where definitions come from. It defaults to `'agents'`, the conventional layout:

- A folder-name `str` (the default`'agents'` ): for the project root (cwd) then the home root, load from`<root>/.agents/<name>/` , falling back to`<root>/.claude/<name>/` when`<root>/.agents/` is absent.
- A sequence of paths loads from exactly those folders, in order.
- `None` disables disk loading, exposing only the explicitly-passed`agents` .

A definition is a markdown file with optional frontmatter:

```
---
name: researcher
description: Researches a topic and reports findings
tools: Read, Grep
---
You research topics. Report your findings, each with a source.
```
- `name` is the delegate name (how the parent refers to it and how it is listed). It falls back to the filename stem when absent.
- `description` drives the prompt listing.
- The markdown body becomes the agent’s instructions.
- `tools` (or`allowed-tools` ) is a comma-separated string or a YAML block list. See “Tools” below.
- `model` and`color` are ignored: the model is inherited from the parent (see below), and`color` has no pyai equivalent.

Frontmatter is read by a small, dependency-free parser limited to those keys (`pyyaml` is not a harness dependency). Full YAML frontmatter is not supported.

Disk agents inherit the parent run’s model by default. Per agent, the caller can override the model and set a thinking/effort level via `agent_overrides`, keyed by the agent’s name:

```
from pydantic_ai_harness.subagents import AgentOverride, SubAgents
SubAgents(
    agent_folders='agents',
    agent_overrides={'researcher': AgentOverride(model='anthropic:claude-sonnet-4-6', effort='high')},
)
```
Every agent the capability builds runs at a minimum thinking-effort floor. `MINIMUM_EFFORT_FLOOR` and the `clamp_effort(level, floor=...)` helper are exported so an orchestrator can apply the same floor to its own agents (that orchestrator-side application is the caller’s responsibility). `clamp_effort` maps `None`/`False` to the floor, leaves `True` (provider-default effort) unchanged, and raises a concrete level below the floor up to it. Effort is applied through pyai’s `ModelSettings.thinking`.

A disk agent gets no tools by default (`inherit_tools` is `False`); set `inherit_tools=True` to expose the parent’s tools to it through the `inherit_tools` mechanism, in which case its `tools` frontmatter is ignored. To map the frontmatter tool names to specific toolsets instead, pass a `tool_resolver`: it receives each tool name (so it can honor entries like `Bash(git:*)`) and returns the toolsets that provide it, or `None` for an unknown name, which is skipped with a warning.

```
from pydantic_ai_harness.subagents import SubAgents
def resolve(tool_name: str):
    return TOOLSETS.get(tool_name)  # -> Sequence[AgentToolset[object]] | None
SubAgents(agent_folders='agents', tool_resolver=resolve)
```
When the same name appears in more than one source, the higher-precedence one wins and the others are skipped with a warning: explicitly-passed `agents` first, then the project folder, then the home folder (and, for an explicit path sequence, earlier paths before later ones). A duplicate name within the explicitly-passed `agents` list is still an error.

```
SubAgents(
    agents=(),             # Sequence[SubAgent[AgentDepsT]] -- each pairs an agent with its run controls
    models={},             # Mapping[str, Model | str | ModelOption] -- per-delegation model menu (off when empty)
    agent_folders='agents',# folder-name str (convention) | Sequence[Path] | None (disable)
    agent_overrides={},    # Mapping[str, AgentOverride] -- per-disk-agent model/effort override
    tool_resolver=None,    # Callable[[str], Sequence[AgentToolset[object]] | None] -- disk-agent tool mapping
    forward_usage=True,    # share the parent's usage with sub-agent runs
    inherit_tools=False,   # expose the parent's own tools to sub-agents (capability tools excluded)
    shared_capabilities=(),# capabilities applied to every sub-agent run
    event_stream_handler=None,  # forwarded to each sub-agent run to stream its events
    tool_name='delegate_task',
    tool_retries=2,        # extra delegate-tool attempts after a sub-agent error before aborting (None inherits the agent default)
    contain_errors=False,  # default for SubAgent.contain_errors: contain an unexpected crash as a bounded retry
)
```
```
SubAgent(
    agent,                 # AbstractAgent[AgentDepsT, Any] -- the child agent to run
    name=None,             # delegate name; defaults to the agent's own `name`
    description=None,      # prompt-listing description; defaults to the agent's own `description`
    models=None,           # Sequence[str] -- menu keys this delegate may run on; the first is its default
    usage_limits=None,     # per-delegation request/token budget (isolated accounting)
    timeout_seconds=None,  # per-delegation wall-clock budget
    max_calls=None,        # max delegations to this sub-agent per parent run
    on_failure=None,       # steering message for soft degradations of this delegate
    contain_errors=None,   # contain an unexpected crash as a bounded retry; None inherits the SubAgents default
)
```
`SubAgents` is not serializable via the [agent spec](/docs/ai/core-concepts/agent-spec/) (it holds live `Agent` instances), so `get_serialization_name()` returns `None`.

- Sub-agents can themselves have `SubAgents` , forming a tree. Share`usage` (the default) and set a`usage_limits` on the top-level run to bound the whole tree.
- Delegations the model issues in parallel run as independent sub-agent runs.

**Bases:** `AbstractCapability[AgentDepsT]`

Let an agent delegate self-contained tasks to named sub-agents.

Exposes a single `delegate_task(agent_name, task)` tool. Each delegation
runs the chosen sub-agent in a fresh, isolated run (it never sees the parent
conversation), and the available sub-agents are listed in the system prompt
as a static, cache-stable instruction.

Sub-agents are passed as a sequence of `SubAgent` entries, each pairing an
agent with its per-delegate run controls (a `usage_limits` budget, a
wall-clock `timeout_seconds`, a per-run `max_calls` budget, an `on_failure`
steering message, and optional `name`/`description` overrides). A delegate’s
name is its `SubAgent.name`, or the agent’s own `name` when unset; two
explicitly-passed delegates resolving to the same name is an error.

Delegations run on the sub-agent’s own model unless a `models` menu is
configured, in which case `delegate_task` also takes a `model` argument naming
one of the menu’s keys, so the parent routes each task to the model that fits
it. A `SubAgent` can restrict which keys it accepts (`SubAgent.models`).

Sub-agents are also loaded from disk by default: each markdown agent definition
under `./.agents/agents/` and `~/.agents/agents/` (or the `.claude/` equivalent)
becomes a delegate, built with the parent’s model. Disk delegates get no tools
by default (`inherit_tools` is `False`); set `inherit_tools=True` to expose the
parent’s tools, or pass a `tool_resolver` to map their frontmatter tool names.
Disk delegates coexist with explicitly-passed ones; explicitly-passed agents take
precedence, then the project folder, then the home folder. A disk delegate whose
name is already taken is skipped with a warning. Configure or disable this with
`agent_folders`; see also `agent_overrides` and `tool_resolver`.

The parent’s `deps` are forwarded to each sub-agent (sub-agents therefore
share the parent’s `AgentDepsT`), and by default the parent’s `usage` is
shared so usage limits apply across the whole agent tree. Optionally, the
parent’s tools can be inherited (`inherit_tools`), extra capabilities can be
applied to every sub-agent run (`shared_capabilities`), and sub-agent events
can be streamed to a handler (`event_stream_handler`).

```
from pydantic_ai import Agent
from pydantic_ai_harness.subagents import SubAgent, SubAgents
researcher = Agent('anthropic:claude-sonnet-4-6', name='researcher', description='Researches topics')
writer = Agent('anthropic:claude-sonnet-4-6', name='writer', description='Writes prose')
orchestrator = Agent(
    'anthropic:claude-opus-4-7',
    capabilities=[SubAgents(agents=[SubAgent(researcher), SubAgent(writer)])],
)
```
The sub-agents to expose, each a `SubAgent` pairing an agent with its
per-delegate run controls. See `SubAgent`. These take precedence over any
disk-loaded agents of the same name.

**Type:** [`Sequence`](https://docs.python.org/3/library/typing.html#typing.Sequence)[`SubAgent`[`AgentDepsT`]] **Default:** `()`

A menu of models the parent can route an individual delegation to, keyed by
the name the parent uses to pick one. Off by default: with no menu the delegate
tool has no `model` argument and every delegation runs the way it always did.

Each value is a model reference, or a `ModelOption` carrying a routing hint and
its own `ModelSettings` (so one key can mean “same model, more thinking”). The
keys and their descriptions are listed in the system prompt, so name them for
the job — `'fast'`, `'deep'` — rather than for the vendor. `SubAgent.models`
restricts which of them a given delegate accepts.

```
from pydantic_ai_harness.subagents import SubAgents
SubAgents(models={'fast': 'anthropic:claude-haiku-4-5', 'deep': 'anthropic:claude-opus-4-7'})
```
**Type:** [`Mapping`](https://docs.python.org/3/library/typing.html#typing.Mapping)[[`str`](https://docs.python.org/3/library/stdtypes.html#str), `Model` | `KnownModelName` | [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `ModelOption`] **Default:** `field(default_factory=(dict[str, 'Model | KnownModelName | str | ModelOption']))`

Where to load markdown agent definitions from, in addition to `agents`.
Defaults to the conventional layout, so constructing the capability auto-loads
a repo’s agent files with no extra configuration.

- a folder-name `str` (the default`'agents'` is the conventional layout): for
the project root (cwd) then the home root, load from`<root>/.agents/<name>/` ,
falling back to`<root>/.claude/<name>/` when`<root>/.agents/` is absent.
- a sequence of paths: load from exactly those folders, in order.
- `None` : disable disk loading entirely (only`agents` are exposed).

Missing folders are skipped. Within a folder every `*.md` file is a candidate.

**Type:** [`str`](https://docs.python.org/3/library/stdtypes.html#str) | [`Sequence`](https://docs.python.org/3/library/typing.html#typing.Sequence)[`Path`] | `None`**Default:** `'agents'`

Per-disk-agent overrides keyed by the agent’s name. An entry can set the
agent’s `model` (otherwise the parent’s model is inherited) and its `effort`
(otherwise the minimum floor). Has no effect on explicitly-passed `agents`.

**Type:** [`Mapping`](https://docs.python.org/3/library/typing.html#typing.Mapping)[[`str`](https://docs.python.org/3/library/stdtypes.html#str), `AgentOverride`] **Default:** `field(default_factory=(dict[str, AgentOverride]))`

Optional override for how a disk agent gets its tools. When set, each tool
name in a definition’s `tools`/`allowed-tools` frontmatter is passed to this
resolver and the returned toolsets are attached to that agent; an unknown name
(resolver returns `None`) is skipped with a warning. When unset, the
frontmatter tool list is ignored and disk agents inherit the parent’s tools
via `inherit_tools` (set `inherit_tools=True` to expose them).

**Type:** `ToolResolver` | `None`**Default:** `None`

If `True`, the parent run’s `usage` is shared with each sub-agent run, so
token usage aggregates and usage limits apply across the whole agent tree.

**Type:** `bool`**Default:** `True`

If `True`, the parent agent’s tools are exposed to each sub-agent run (the
delegate tool itself is filtered out, so sub-agents can’t recurse into
further delegation). Off by default to avoid silently widening sub-agent access.

**Type:** `bool`**Default:** `False`

Capabilities applied to every sub-agent run, in addition to whatever each sub-agent already has.

**Type:** [`Sequence`](https://docs.python.org/3/library/typing.html#typing.Sequence)[[`AgentCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AgentCapability)[`AgentDepsT`]] **Default:** `()`

If set, this handler is passed to each sub-agent run, so the sub-agent’s
model-streaming and tool events surface to the caller. The handler receives
the sub-agent’s own `RunContext` and event stream.

**Type:** `EventStreamHandler`[`AgentDepsT`] | `None`**Default:** `None`

Name of the delegate tool exposed to the model.

**Type:** `str`**Default:** `'delegate_task'`

Retries for the delegate tool — how many extra attempts it gets after a
sub-agent error before the parent run aborts. A sub-agent failure (e.g. it
exhausts its own output retries) surfaces to the parent as a tool retry it
can react to by re-delegating with a corrected task. The retry counter
resets after any successful delegation, so this bounds consecutive failures,
not total ones. Defaults to `2` (pydantic-ai’s per-tool default is `1`) so a
repeated flaky sub-agent does not abort the parent run on its first repeat;
set `None` to inherit the parent agent’s default tool retries instead.

Default for `SubAgent.contain_errors`: whether an unexpected sub-agent crash
is caught and returned to the parent as a bounded `ModelRetry` instead of
aborting the parent run. Off by default, so a crash propagates. Any `SubAgent`
can override this per delegate. See `SubAgent.contain_errors` for the
containment contract and what always propagates regardless.

**Type:** `bool`**Default:** `False`

`@async`

```
def wrap_run(
    ctx: RunContext[AgentDepsT],
    *,
    handler: WrapRunHandler,
) -> AgentRunResult[Any]
```
Run the parent agent, then drop this run’s delegation counts so they don’t accumulate.

```
def get_instructions() -> AgentInstructions[AgentDepsT] | None
```
Static, cache-stable listing of the available sub-agents and models.

`AgentInstructions`[`AgentDepsT`] | `None`

```
def get_toolset() -> AgentToolset[AgentDepsT] | None
```
Toolset providing the delegate tool, or `None` when no sub-agents are configured.

[`AgentToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AgentToolset)[`AgentDepsT`] | `None`

`@classmethod`

```
def get_serialization_name(cls) -> str | None
```
Not spec-serializable — the capability holds live `Agent` instances.

**Bases:** `Generic[AgentDepsT]`

One delegate: a child agent plus its per-delegate run controls.

Pass a sequence of these as `SubAgents(agents=[...])`. The delegate’s name —
how the parent model refers to it, and how it is listed in the system prompt —
is `name` when set, otherwise the agent’s own `name`. An agent with neither is
rejected by `SubAgents`.

Every control below is optional; an unset field leaves the corresponding
behaviour at the `SubAgents` default.

The agent that runs when this delegate is invoked.

**Type:** `AbstractAgent`[`AgentDepsT`, [`Any`](https://docs.python.org/3/library/typing.html#typing.Any)]

Name the parent model uses to delegate to this agent. Defaults to the
agent’s own `name` when unset.

**Type:** [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `None`**Default:** `None`

Description for the system-prompt listing. Defaults to the agent’s own
`description` when unset; a delegate with neither is listed by name alone.

**Type:** [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `None`**Default:** `None`

Which of `SubAgents.models` this delegate may run on, as menu keys, and
which one it runs on by default: the first key listed. Leave it unset to let
the parent pick any configured option and to fall back to the delegate’s own
model when it picks none. Set it to pin a delegate to one option
(`models=['fast']`) or to bound an expensive delegate to a subset. Naming a key
the menu does not define is an error.

**Type:** [`Sequence`](https://docs.python.org/3/library/typing.html#typing.Sequence)[[`str`](https://docs.python.org/3/library/stdtypes.html#str)] | `None`**Default:** `None`

Request/token budget for one delegation. When set, the child runs with
its own usage accounting so the budget counts only the child’s own requests
and tokens (not the parent’s or siblings’), even when `forward_usage=True`.
The tradeoff: that child’s tokens no longer aggregate into the parent’s
`usage`. Hitting this budget is a soft outcome (steering message), not a
run-stopping `UsageLimitExceeded`.

**Type:** [`UsageLimits`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.UsageLimits) | `None`**Default:** `None`

Wall-clock budget for one delegation. When the child exceeds it, the run is cancelled and the parent gets a soft steering message instead of hanging on the child.

**Type:** [`float`](https://docs.python.org/3/library/functions.html#float) | `None`**Default:** `None`

Maximum number of delegations to this sub-agent per parent run. Once reached, further delegations return a soft budget-exhausted message without running the child.

**Type:** [`int`](https://docs.python.org/3/library/functions.html#int) | `None`**Default:** `None`

Steering message returned to the parent for any soft degradation of this
delegate (timeout, child failure, usage budget reached, call budget
exhausted), in place of the built-in default. Setting it also makes child
failures soft: a child error returns this message as a normal tool result
instead of raising a parent `ModelRetry`.

**Type:** [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `None`**Default:** `None`

Whether an unexpected sub-agent crash is contained instead of aborting the
parent run. When `True`, an exception the child raises that is not an expected
soft degradation (a provider `ModelAPIError`/`FallbackExceptionGroup`, a plain
`ValueError` from a bad tool argument, etc.) is caught and returned to the parent
as a bounded `ModelRetry`, so one delegate crash cannot kill the whole run. It
stays loud: the exception rides the retry message and is logged, and
`tool_retries` still bounds consecutive crashes into an abort. Cancellation, a
shared usage-limit, pydantic-ai control-flow signals, and `UserError` always
propagate regardless. Unset inherits `SubAgents.contain_errors` (default off).
Orthogonal to `on_failure`, which only sets the message for expected soft
degradations; a contained crash always raises the loud `ModelRetry`.

**Type:** [`bool`](https://docs.python.org/3/library/functions.html#bool) | `None`**Default:** `None`

The delegate’s name: `name` if set, else the agent’s own `name`.

One entry on the model menu, as a model plus how it should run.

Pass bare model references when the menu keys speak for themselves, and a
`ModelOption` when an entry needs a routing hint or its own settings:

```
from pydantic_ai.settings import ModelSettings
from pydantic_ai_harness.subagents import ModelOption, SubAgents
SubAgents(
    models={
        'fast': 'anthropic:claude-haiku-4-5',
        'deep': ModelOption(
            'anthropic:claude-opus-4-7',
            description='hard reasoning, multi-file changes',
            settings=ModelSettings(thinking='xhigh'),
        ),
    },
)
```
The model a delegation routed to this option runs on.

**Type:** `Model` | `KnownModelName` | `str`

What this option is for, listed in the prompt next to the key so the parent can route on task difficulty rather than on model names alone.

**Type:** [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `None`**Default:** `None`

Settings for a delegation routed to this option — thinking effort,
temperature, and so on. They merge over the sub-agent’s own `model_settings`,
which keeps whatever the sub-agent set and is not overridden here.

**Type:** [`ModelSettings`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings) | `None`**Default:** `None`

Per-agent override for a disk-loaded sub-agent, keyed by the agent’s name.

Both fields are optional. An unset `model` inherits the parent run’s model; an
unset `effort` runs at the capability’s minimum effort floor (see
`clamp_effort`).

Model to run this disk agent with, in place of inheriting the parent’s.

**Type:** `Model` | `KnownModelName` | [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `None`**Default:** `None`

Thinking/reasoning level for this disk agent. Raised to at least the floor.

**Type:** `ThinkingLevel` | `None`**Default:** `None`

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/subagents
