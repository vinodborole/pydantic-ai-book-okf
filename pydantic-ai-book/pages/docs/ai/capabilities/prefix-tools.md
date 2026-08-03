---
type: Web Page
title: Prefix Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/prefix-tools
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Prefix Tools

[`PrefixTools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrefixTools) is a [capability](/docs/ai/capabilities/overview) that wraps another capability and prefixes all of its tool names, useful for namespacing when composing multiple capabilities that might have conflicting tool names:

Every [`AbstractCapability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability) has a convenience method [`prefix_tools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.AbstractCapability.prefix_tools) that returns a [`PrefixTools`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.PrefixTools) wrapper:

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/prefix-tools
