---
type: Web Page
title: Coder | Pydantic Docs
description: A complete Pydantic AI coding-agent harness assembled from transparent
  capabilities.
resource: https://pydantic.dev/docs/ai/harness/coder
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Coder

`Coder` gives a Pydantic AI agent a complete, opinionated stack for working in a local codebase.
It is a regular [combined capability](https://pydantic.dev/docs/ai/capabilities/custom/#composition-and-middleware-semantics) made from the [capabilities](https://pydantic.dev/docs/ai/capabilities/overview/) below, so you can use it as-is or take it apart.

While Pydantic AI Harness is on 0.x releases, the API may change between minor releases; when it does, deprecation warnings and release-note migration guidance tell you (or your agent) exactly how to upgrade. See the [version policy](/docs/ai/harness/#version-policy).

Hand the agent a task directly:

```
from pydantic_ai import Agent
from pydantic_ai_harness import Coder
agent = Agent('anthropic:claude-fable-5', capabilities=[Coder('.')])
result = agent.run_sync('Find out why tests/test_parser.py fails and fix the bug it caught.')
print(result.output)
#> Found it: `parse()` returned None on empty input instead of raising. Fixed in src/parser.py; tests pass now.
```
The same agent works with every Pydantic AI interface: [`agent.to_cli_sync()`](https://pydantic.dev/docs/ai/cli/) starts an interactive chat in your terminal, and [`agent.to_web()`](https://pydantic.dev/docs/ai/web/) serves a browser chat UI.

Or skip the file entirely and run the exported [`coder_agent`](#api-reference) with [`clai`](https://pydantic.dev/docs/ai/cli/#custom-agents) (the Pydantic AI CLI), via [`uvx`](https://docs.astral.sh/uv/guides/tools/):

It is literally these capabilities combined, in this order:

- [`FileSystem`](/docs/ai/harness/filesystem/) : read, write, edit, and search tools rooted at the workspace, path-traversal and symlink safe
- [`Shell`](/docs/ai/harness/shell/) : allowlisted commands rooted at the workspace (a guardrail, not a security boundary), with common LLM provider API-key variables filtered from inherited command environments
- [`RepoContext`](/docs/ai/harness/repo-context/) : repository instructions and structure
- [`Planning`](/docs/ai/harness/planning/) : a plan the agent creates and keeps current during multi-step work
- [`SubAgents`](/docs/ai/harness/subagents/) : delegation, with a read-only`explorer` sub-agent by default
- [`ClearToolResults`](/docs/ai/harness/compaction/) : clears stale tool results at 70% of the model context window
- [`WarnNearLimits`](/docs/ai/harness/compaction/) : warns the agent at 90% of the model context window
- [`ToolOutputLimits`](/docs/ai/harness/tool-output-limits/) : bounds how much context any single tool result can consume

Pass `subagents=[]` to disable delegation, or supply your own `SubAgent` entries.

`Coder` ships with **no default instructions**: modern models don’t need procedural coaching (“work step by step”, “run the tests”), and each composed capability already contributes its own tool guidance. Pass `instructions='...'` to add your own (identity, tone, or house rules) and it becomes a regular instructions capability at the front of the composition. The exported `coder_agent` separately carries the identity instruction `You are a coding agent built on Pydantic AI.`

The command allowlist is a guardrail against accidents, not a security boundary. Validation checks only the first token, and allowlisted commands such as `python`, `git`, `uv`, and `make` can spawn arbitrary processes, so a model that wants to work around the allowlist can. For untrusted work, run the agent inside an OS-level sandbox such as [`ModalSandbox`](/docs/ai/harness/modal-sandbox/) or a container.

Other capabilities pair well with `Coder`; add them alongside it in `capabilities`:

- [Web Search](https://pydantic.dev/docs/ai/capabilities/web-search/) and[Web Fetch](https://pydantic.dev/docs/ai/capabilities/web-fetch/) (core): look up docs and error messages on the web
- [Skills](/docs/ai/harness/skills/) : reusable procedure documents the agent loads on demand
- [Memory](/docs/ai/harness/memory/) : persistent memory across conversations
- [Conversation Search](/docs/ai/harness/conversation-search/) : let the agent search earlier sessions
- [Guardrails](/docs/ai/harness/guardrails/) : validate what the agent does before and after it acts
- [Dynamic Workflow](/docs/ai/harness/dynamic-workflow/) : let the agent author multi-step workflows; best activated on demand

```
from pydantic_ai import Agent
from pydantic_ai.capabilities import WebSearch
from pydantic_ai_harness import Coder, Memory
from pydantic_ai_harness.memory import FileStore
agent = Agent(
    'anthropic:claude-fable-5',
    capabilities=[
        Coder(),
        WebSearch(),  # look up docs and error messages on the web
        Memory(FileStore('.agent-memory')),  # remembers across sessions
    ],
)
```
Add [`Skills('skills')`](/docs/ai/harness/skills/) to the list once you have a `skills/` directory of `SKILL.md` procedures to point it at (it validates the directory eagerly, and needs the `skills` extra).

To remove or replace one of the built-in components instead, start from the blown-out form below and adjust the list.

This is the exact agent the exported `coder_agent` gives you (plus an explicit model), written out block by block:

```
from pathlib import Path
from pydantic_ai import Agent
from pydantic_ai_harness import (
    ClearToolResults,
    FileSystem,
    LLM_API_KEY_ENV_PATTERNS,
    Planning,
    RepoContext,
    Shell,
    SubAgent,
    SubAgents,
    ToolOutputLimits,
    WarnNearLimits,
)
allowed_commands = [
    'git', 'rg', 'grep', 'find', 'ls', 'cat', 'sed', 'head', 'tail',
    'python', 'uv', 'pytest', 'ruff', 'make',
]
explorer = SubAgent(
    Agent(
        name='explorer',
        description='Explore the codebase and answer questions without modifying anything',
        instructions='Answer with concrete paths and evidence.',
        capabilities=[
            FileSystem('.', read_only=True),
            RepoContext(workspace_dir=Path('.')),
        ],
    )
)
agent = Agent(
    'anthropic:claude-fable-5',
    name='coder',
    instructions='You are a coding agent built on Pydantic AI.',
    capabilities=[
        FileSystem('.'),  # read/write/edit/search, path-traversal safe
        Shell(  # allowlisted commands, LLM API keys stripped from their environment
            cwd='.',
            allowed_commands=allowed_commands,
            denied_env_patterns=LLM_API_KEY_ENV_PATTERNS,
        ),
        RepoContext(workspace_dir=Path('.')),  # loads AGENTS.md/CLAUDE.md + repo structure
        Planning(),  # structured task plans the model maintains
        SubAgents(agents=[explorer], agent_folders=None),  # delegate exploration off the main context
        ClearToolResults(max_fraction=0.7),  # clears old tool results near the limit
        WarnNearLimits(max_context_fraction=0.9),  # warns the model before it hits limits
        ToolOutputLimits(),  # bounds oversized tool results
    ],
)
```
See the [source](https://github.com/pydantic/pydantic-ai-harness/tree/main/pydantic_ai_harness/coder/).

**Bases:** `CombinedCapability[AgentDepsT]`

A complete coding-agent harness built as a regular combined capability.

See the class definition and [Coder docs](https://pydantic.dev/docs/ai/harness/coder/) for the exact composition.

It ships with no default instructions: modern models don’t need procedural coaching, and the composed capabilities
contribute their own tool guidance. Pass `instructions=` to add your own.

The command allowlist is a guardrail against accidents, not a security boundary. Validation checks only the first
token, and allowed commands such as `python`, `git`, `uv`, and `make` can spawn arbitrary processes. Run untrusted
work in an OS-level sandbox such as `ModalSandbox` or a container.

Model-less coding agent for CLIs that load `module:variable` targets.

**Default:** `Agent(name='coder', instructions='You are a coding agent built on Pydantic AI.', capabilities=[Coder()])`

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/coder
