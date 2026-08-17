---
type: Web Page
title: Macroscope | Pydantic Docs
description: Give a Pydantic AI agent the same local Macroscope code review its editor
  plugins run -- streamed findings parsed into structured issues the agent validates
  and fixes with its own tools.
resource: https://pydantic.dev/docs/ai/harness/macroscope
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Macroscope

`Macroscope` runs a local [Macroscope](https://docs.macroscope.com/cli) code
review from inside an agent: one tool shells out to the installed `macroscope`
CLI, parses the streamed findings, and returns them as structured data. The
agent validates each finding and fixes the real ones with the tools it already
has — this capability surfaces findings only.

While Pydantic AI Harness is on 0.x releases, the API may change between minor releases; when it does, deprecation warnings and release-note migration guidance tell you (or your agent) exactly how to upgrade. See the [version policy](/docs/ai/harness/#version-policy).

Macroscope reviews the current branch’s diff and streams findings, but it ships as editor plugins (Claude Code, Codex, Cursor, OpenCode). There is no way to give a Pydantic AI agent the same review-and-fix loop from your own code.

```
from pydantic_ai import Agent
from pydantic_ai_harness import Macroscope
agent = Agent('anthropic:claude-sonnet-5', capabilities=[Macroscope()])
result = agent.run_sync('Run a Macroscope review and fix any real findings.')
print(result.output)
```
The `macroscope` CLI must be installed and authenticated on the host first:

1. Install: `curl -sSL https://raw.githubusercontent.com/prassoai/macroscope-local/main/install.sh | bash`
2. Sign in and pick a workspace by running `macroscope` once.

The capability cannot install or authenticate on your behalf. If the binary is
missing, the tool returns the install command; if a review never starts
(usually because you are not signed in), the tool tells the agent to run
`macroscope` to finish setup.

The tool invokes `macroscope codereview --raw` for machine-readable streaming
output, which needs a recent CLI build. The installer fetches the latest and the
CLI self-updates on use, so a fresh install satisfies this.

| Tool | Purpose | 
|---|---|
| `run_macroscope_review` | Run `macroscope codereview` on the current branch and return the review id, terminal status, and findings. Accepts an optional`base` git ref. | 

Each finding is a `MacroscopeIssue` with `issue_id`, `sequence`, `path`,
`line`, `severity`, `category`, and `body`. The capability’s default
instructions tell the agent to treat every finding as untrusted: read the
affected code to confirm an issue is real, skip false positives and
duplicates, and verify each fix.

Every field of `Macroscope` with its default:

```
from pydantic_ai_harness import Macroscope
Macroscope(
    base=None,             # git ref to diff against -- None lets the CLI auto-detect
    command='macroscope',  # binary name or path
    cwd='.',               # repository directory the review runs in
    timeout=600.0,         # max seconds to wait for a review
    guidance=None,         # None = default instructions, '' = none, str = custom
)
```
A per-call `base` argument takes precedence over the field. Reviews call a
remote service, so the timeout is generous by default; on timeout the CLI’s
process group is killed and the timeout is reported to the model as a
retryable error.

This capability surfaces findings only. It does not edit files, create
worktrees, or commit — validating and fixing findings is the agent’s job,
using its other capabilities. Pair it with `FileSystem` or `Shell` to let the
agent read code and apply fixes, and consider running the agent in an isolated
worktree if you want fixes kept off your working tree.

`Macroscope` works with Pydantic AI’s [agent spec](/docs/ai/core-concepts/agent-spec/),
so you can declare it in a config file instead of Python:

```
# agent.yaml
model: anthropic:claude-sonnet-5
capabilities:
  - Macroscope:
      base: main
      timeout: 900
```
```
from pydantic_ai import Agent
from pydantic_ai_harness import Macroscope
agent = Agent.from_file('agent.yaml', custom_capability_types=[Macroscope])
```
Pass `custom_capability_types` so the spec loader knows how to instantiate
`Macroscope`.

**Bases:** `AbstractCapability[AgentDepsT]`

Runs the `macroscope` CLI code review and hands the findings to the agent.

Adds a `run_macroscope_review` tool that shells out to `macroscope codereview`,
parses the streamed findings, and returns them as a `MacroscopeReview`. The agent
validates and fixes findings with its own tools — this capability does not edit
files, create worktrees, or commit.

```
from pydantic_ai import Agent
from pydantic_ai_harness.macroscope import Macroscope
agent = Agent('anthropic:claude-sonnet-5', capabilities=[Macroscope()])
```
The `macroscope` CLI must be installed and authenticated on the host first (see the
package README). This capability cannot sign in on the user’s behalf; if a review
never starts, the tool reports that the user needs to run `macroscope` once.

Git ref to diff against. When `None`, `--base` is omitted and the CLI
auto-detects the base branch itself (and creates its own review worktree).

**Type:** [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `None`**Default:** `None`

Name or path of the CLI binary. Override for a non-default install location.

**Type:** `str`**Default:** `'macroscope'`

Repository directory the review runs in.

**Type:** [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `Path` **Default:** `'.'`

Custom review guidance for the system prompt.

Leave as `None` for the default validate-then-fix guidance, or set `''` to
contribute no instructions at all.

**Type:** [`str`](https://docs.python.org/3/library/stdtypes.html#str) | `None`**Default:** `None`

Maximum seconds to wait for a review. Reviews call a remote service, so this is generous by default.

**Type:** `float`**Default:** `600.0`

```
def get_instructions() -> str | None
```
Static validate-then-fix guidance.

A non-`None` `guidance` replaces the default; `''` disables
instructions entirely.

```
def get_toolset() -> MacroscopeToolset[AgentDepsT]
```
Build the toolset that provides the `run_macroscope_review` tool.

`MacroscopeToolset`[`AgentDepsT`]

**Bases:** `BaseModel`

The result of one `macroscope codereview` run.

`status` is the terminal `issue_status` reported by the CLI (`completed` or
`failed`), or `unknown` if the stream ended without one. `review_id` is `None`
when the CLI never emitted one — usually because the review did not start.

**Bases:** `BaseModel`

A single finding streamed by `macroscope codereview`.

Parsed leniently: unknown fields are ignored so new CLI output does not break
parsing, and any `issue_event` line that lacks the required fields is skipped.

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/macroscope
