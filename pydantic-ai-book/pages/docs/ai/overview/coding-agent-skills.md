---
type: Web Page
title: Coding Agent Skills | Pydantic Docs
resource: https://pydantic.dev/docs/ai/overview/coding-agent-skills
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Coding Agent Skills

If you’re building Pydantic AI applications with a coding agent, you can install the Pydantic AI skill from the `pydantic/skills` repository to give your agent up-to-date framework knowledge.

Agent skills are packages of instructions and reference material that coding agents load on demand. With the skill installed, coding agents have access to Pydantic AI patterns, architecture guidance, and common task references covering tools, capabilities, structured output, streaming, testing, multi-agent delegation, hooks, and agent specs.

Install the official Pydantic AI plugin from the Anthropic marketplace, which is available by default:

As an alternative, you can install from the `pydantic/skills` marketplace, which bundles the Pydantic AI skill alongside other Pydantic-maintained skills:

Install the Pydantic AI skill using the skills CLI:

This works with 30+ agents via the agentskills.io standard, including Claude Code, Codex, Cursor, and Gemini CLI.

Pydantic AI also ships its skill bundled with the package, so you can install it directly from your project’s dependencies via library-skills.io:

The `--all` flag is required because the skill is bundled in `pydantic-ai-slim`, which is a transitive dependency of the `pydantic-ai` meta-package. Without it, `library-skills` only scans direct dependencies and won’t discover the skill.

Add `--claude` to also install into `.claude/skills/` alongside the default `.agents/skills/` directory, since Claude Code doesn’t read from `.agents/`.

- `pydantic/skills`: source repository
- agentskills.io: the open standard for agent skills
- library-skills.io: install agent skills bundled with your project’s dependencies

# Citations

1. Source page: https://pydantic.dev/docs/ai/overview/coding-agent-skills
