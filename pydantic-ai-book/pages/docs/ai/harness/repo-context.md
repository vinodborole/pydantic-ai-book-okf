---
type: Web Page
title: Repo Context | Pydantic Docs
description: Discover and load a repo's accumulated coding-assistant context engineering
  -- instruction files, skills, sub-agents, and hooks.
resource: https://pydantic.dev/docs/ai/harness/repo-context
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Repo Context

`RepoContext` discovers and loads a repo’s accumulated coding-assistant context engineering (CE): the instruction files (`CLAUDE.md`/`AGENTS.md`) scattered across the tree and the assets under `.claude`/`.agents`/`.codex`/`.grok` (skills, sub-agents, hooks).

The API may change between releases. Where practical, breaking changes ship with a deprecation warning.

A repo accumulates CE for whatever coding assistant worked in it: instruction files (`CLAUDE.md`/`AGENTS.md`) scattered across the tree, and assets under `.claude`/`.agents`/`.codex`/`.grok` (skills, sub-agents, hooks). An agent that loads only the top-level instruction file misses the ancestor context and has no idea the rest of the setup exists, so it can neither honor it nor translate it.

`RepoContext` bundles three strategies, each independently toggleable. Construct it with `RepoContext(...)` in an `Agent`’s `capabilities`, anchored at the deepest directory the agent works in:

```
from pathlib import Path
from pydantic_ai import Agent
from pydantic_ai_harness.repo_context import RepoContext
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    capabilities=[RepoContext(workspace_dir=Path('.'), home_dir=Path.home())],
)
result = agent.run_sync('Summarize the coding-assistant setup in this repo.')
print(result.output)
```
Loads `CLAUDE.md`/`AGENTS.md` from `workspace_dir` and every ancestor up to `home_dir` (inclusive). Precedence is ancestor-first, workspace-last: broadest context first, most specific last. Files are deduped by resolved real path and by content hash, so a symlinked `AGENTS.md -> CLAUDE.md` or two ancestors sharing identical content load once.

When `home_dir` is `None` (the default), only `workspace_dir` is scanned — no walk-up. Pass `home_dir=Path.home()` to walk up to your home directory.

Exposes one tool, `inventory_agent_context()`, that reports where the repo’s CE assets live — the `.claude`/`.agents`/`.codex`/`.grok` roots and, within each, the `skills/` (`SKILL.md`), `agents/` (`.md`), and `settings.json` (hooks) it contains. It returns a structured `AgentContextInventory`; it locates assets and does not parse them, leaving translation to the orchestrator.

Rename the tool with `inventory_tool_name`, or scope which roots it scans with `asset_roots`.

When the model lists or reads a directory, surface that directory’s `CLAUDE.md`/`AGENTS.md`. This couples to the host’s list/read tools, so it is opt-in and configurable:

```
from pathlib import Path
from pydantic_ai import Agent
from pydantic_ai_harness import FileSystem
from pydantic_ai_harness.repo_context import RepoContext
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    capabilities=[
        FileSystem(root_dir='.'),
        RepoContext(
            workspace_dir=Path('.'),
            nested_traversal=True,
            traversal_tool_names=frozenset({'list_directory', 'read_file'}),  # the FileSystem tool names to hook
            traversal_path_arg='path',                                   # the path arg key
            nested_inject='pointer',                                     # or 'contents'
        )
    ],
)
```
`nested_inject='pointer'` (default) appends a one-line note pointing at the file; `'contents'` inlines the file body. Each directory is surfaced at most once per run.

Injecting file contents into the system prompt costs prompt-cache stability: a changed prefix re-bills the whole cached region. `RepoContext` keeps the two cache-relevant paths separate:

- Strategy 1 reads its files once at run start and injects them as static system instructions, so the cached prefix stays byte-identical across turns.
- Strategy 3 is volatile (it depends on which directory was just touched), so its note is appended to the tool result in the message tail — never to the system prompt — and cannot invalidate the cached prefix.

```
RepoContext(
    workspace_dir,                  # Path -- the deepest dir the agent works in (required)
    home_dir=None,                  # Path | None -- shallowest dir to stop walk-up at, inclusive
    filenames=('CLAUDE.md', 'AGENTS.md'),
    autoload_instructions=True,     # Strategy 1
    expose_inventory_tool=True,     # Strategy 2
    inventory_tool_name='inventory_agent_context',
    nested_traversal=False,         # Strategy 3
    nested_inject='pointer',        # 'pointer' | 'contents'
    traversal_tool_names=frozenset({'list_directory', 'read_file'}),
    traversal_path_arg='path',
    asset_roots=('.claude', '.agents', '.codex', '.grok'),
)
```
`RepoContext` locates and loads CE; it does not parse skill/sub-agent frontmatter or hook bodies, and it does not rewrite or translate assets. Strategy 1 reads its files once per run, so mid-run edits to those files are not reloaded.

**Bases:** `AbstractCapability[AgentDepsT]`

Discover and load a repo’s accumulated coding-assistant context engineering.

Three strategies, each independently toggleable:

1. 
Walk-up instruction autoload ( `autoload_instructions` , on by default):
load`CLAUDE.md` /`AGENTS.md` from`workspace_dir` and every ancestor up
to`home_dir` , deduped, ancestor-first. These are read once at run start
and injected as**static system instructions** via`get_instructions` , so
they stay in the cached prefix and never re-read per turn.
2. 
Asset inventory ( `expose_inventory_tool` , on by default): a tool that
reports where the repo’s CE assets live (`.claude` /`.agents` /`.codex` /`.grok` and their`skills/` ,`agents/` ,`settings.json` ). It locates
assets; it does not parse them.
3. 
Nested-on-traversal ( `nested_traversal` , off by default): when the model
lists or reads a directory (via a tool named in`traversal_tool_names` ),
surface that directory’s`CLAUDE.md` /`AGENTS.md` . The note is appended to
the**tool result** (message tail), not to system instructions, so it
does not invalidate the cached prefix.`nested_inject='pointer'` (default)
appends a one-line pointer;`'contents'` inlines the file body.

Cache note: injecting file contents into the system prompt costs prompt-cache stability. Strategy 1 is safe because its files are static; the volatile Strategy 3 content rides in the message tail instead.

```
from pathlib import Path
from pydantic_ai import Agent
from pydantic_ai_harness.repo_context import RepoContext
agent = Agent(
    'anthropic:claude-sonnet-4-6',
    capabilities=[RepoContext(workspace_dir=Path('.'), home_dir=Path.home())],
)
```
The deepest directory the agent works in. The walk-up and asset scan are anchored here.

**Type:** `Path`

The shallowest directory to stop the walk-up at, inclusive. `None` (the
default) scans only `workspace_dir` — no walk-up.

**Type:** `Path` | `None`**Default:** `None`

Instruction filenames to look for, in within-directory precedence order.

**Type:** [`Sequence`](https://docs.python.org/3/library/typing.html#typing.Sequence)[[`str`](https://docs.python.org/3/library/stdtypes.html#str)] **Default:** `('CLAUDE.md', 'AGENTS.md')`

Strategy 1: load instruction files into the system prompt.

**Type:** `bool`**Default:** `True`

Strategy 2: expose the asset-inventory tool.

**Type:** `bool`**Default:** `True`

Name of the inventory tool exposed to the model.

**Type:** `str`**Default:** `'inventory_agent_context'`

Strategy 3: surface a directory’s instruction file when the model lists or reads that directory. Off by default — it couples to the list/read tools.

**Type:** `bool`**Default:** `False`

For Strategy 3: append a one-line `pointer`, or inline the file `contents`.

**Type:** [`Literal`](https://docs.python.org/3/library/typing.html#typing.Literal)[‘pointer’, ‘contents’] **Default:** `'pointer'`

Tool names that trigger Strategy 3. Override to match the host’s list/read
tools (e.g. `frozenset({'list_dir', 'read_file'})`).

**Type:** [`frozenset`](https://docs.python.org/3/library/stdtypes.html#frozenset)[[`str`](https://docs.python.org/3/library/stdtypes.html#str)] **Default:** `frozenset({'list_directory', 'read_file'})`

The tool argument key holding the listed/read path.

**Type:** `str`**Default:** `'path'`

Root directories the inventory tool scans, relative to `workspace_dir`.

**Type:** [`Sequence`](https://docs.python.org/3/library/typing.html#typing.Sequence)[[`str`](https://docs.python.org/3/library/stdtypes.html#str)] **Default:** `('.claude', '.agents', '.codex', '.grok')`

`@async`

```
def for_run(ctx: RunContext[AgentDepsT]) -> RepoContext[AgentDepsT]
```
Return a fresh per-run instance with isolated traversal/cache state.

`RepoContext`[`AgentDepsT`]

```
def get_instructions() -> AgentInstructions[AgentDepsT] | None
```
Static, cache-stable instructions: loaded files plus the inventory hint.

`AgentInstructions`[`AgentDepsT`] | `None`

```
def get_toolset() -> AgentToolset[AgentDepsT] | None
```
The asset-inventory toolset, or `None` when the tool is disabled.

[`AgentToolset`](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AgentToolset)[`AgentDepsT`] | `None`

`@async`

```
def after_tool_execute(
    ctx: RunContext[AgentDepsT],
    *,
    call: ToolCallPart,
    tool_def: ToolDefinition,
    args: dict[str, Any],
    result: Any,
) -> Any
```
Strategy 3: append a directory’s instruction file to a list/read result.

`@classmethod`

```
def get_serialization_name(cls) -> str | None
```
Serialization name for agent-spec support.

**Bases:** `BaseModel`

A map of where a repo’s CE assets live, for an orchestrator to read or translate.

**Bases:** `BaseModel`

Where CE assets live under a single root directory (e.g. `.claude`).

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/repo-context
