---
type: Web Page
title: Coding Agent Skills | Pydantic Docs
resource: https://pydantic.dev/docs/ai/overview/coding-agent-skills
timestamp: '2026-07-27T09:59:11.298696+00:00'
---

# Coding Agent Skills

If you’re building Pydantic AI applications with a coding agent, you can install the Pydantic AI skill from the [ pydantic/skills](https://github.com/pydantic/skills) repository to give your agent up-to-date framework knowledge.

[Agent skills](https://agentskills.io) are packages of instructions and reference material that coding agents load on demand. With the skill installed, coding agents have access to Pydantic AI patterns, architecture guidance, and common task references covering [tools](/docs/ai/tools-toolsets/tools), [capabilities](/docs/ai/capabilities/overview), [structured output](/docs/ai/core-concepts/output), [streaming](/docs/ai/core-concepts/agent#streaming-events-and-final-output), [testing](/docs/ai/guides/testing), [multi-agent delegation](/docs/ai/guides/multi-agent-applications), [hooks](/docs/ai/core-concepts/hooks), and [agent specs](/docs/ai/core-concepts/agent-spec).

Install the [official Pydantic AI plugin](https://claude.com/plugins/pydantic-ai) from the Anthropic marketplace, which is available by default:

As an alternative, you can install from the [ pydantic/skills](https://github.com/pydantic/skills) marketplace, which bundles the Pydantic AI skill alongside other Pydantic-maintained skills:

Install the Pydantic AI skill using the [skills CLI](https://github.com/vercel-labs/skills):

This works with 30+ agents via the [agentskills.io](https://agentskills.io) standard, including Claude Code, Codex, Cursor, and Gemini CLI.

Pydantic AI also ships its skill bundled with the package, so you can install it directly from your project’s dependencies via [library-skills.io](https://library-skills.io):

The `--all` flag is required because the skill is bundled in `pydantic-ai-slim`, which is a transitive dependency of the `pydantic-ai` meta-package. Without it, `library-skills` only scans direct dependencies and won’t discover the skill.

Add `--claude` to also install into `.claude/skills/` alongside the default `.agents/skills/` directory, since Claude Code doesn’t read from `.agents/`.

- `pydantic/skills`
- [agentskills.io](https://agentskills.io): the open standard for agent skills
- [library-skills.io](https://library-skills.io): install agent skills bundled with your project’s dependencies

# Citations

1. Source page: https://pydantic.dev/docs/ai/overview/coding-agent-skills
