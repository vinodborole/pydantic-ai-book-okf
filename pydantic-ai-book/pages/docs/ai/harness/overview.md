---
type: Web Page
title: Overview | Pydantic Docs
resource: https://pydantic.dev/docs/ai/harness/overview
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Overview

**Pydantic AI Harness** is the official capability library for Pydantic AI — the batteries for your agent.

Pydantic AI core ships the agent loop, model providers, the capabilities/hooks abstraction, and two kinds of capabilities:

- **Capabilities that require model or framework support**— anything backed by provider native tools (like image generation), provider-specific APIs (like compaction via the OpenAI or Anthropic APIs), or deep agent graph integration. These go hand-in-hand with model class code and need to ship together.
- **Capabilities that are fundamental to the agent experience**— things nearly every agent benefits from, like web search, tool search, and thinking. These feel like qualities of the agent itself, not accessories.

**Pydantic AI Harness** is where everything else lives: standalone capabilities that make specific categories of agents powerful, or that are still finding their final shape. Context management, memory, guardrails, file system access, code execution, multi-agent orchestration — these are the building blocks you pick and choose based on what your agent needs to do.

The harness is also where new capabilities *start*. It ships as a separate package so capabilities can iterate faster without the strict backward-compatibility requirements of core. As a capability stabilizes and proves itself broadly essential, it can graduate into core — code mode is an early candidate.

Many capabilities benefit from a “fall up” pattern: they typically start as a local implementation that works with every model, then gain provider-native support that uses the provider’s built-in API when available — auto-switching between the two. This is how web search, web fetch, and image generation already work in core, and the same approach is coming for skills, code mode, and context compaction.

See the capability matrix on the harness README for the full list of what’s available and what’s coming, along with community alternatives.

Capability contributions should go to the harness repo, not to pydantic-ai. The capabilities abstraction gives contributions clear boundaries, which makes them easier to review. See Contributing for details.

You can also publish capabilities as standalone packages. See Building custom capabilities for the API and Publishing capability packages for packaging guidance.

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/overview
