---
type: Web Page
title: Extensibility | Pydantic Docs
resource: https://pydantic.dev/docs/ai/guides/extensibility
timestamp: '2026-07-20T09:23:04.251034+00:00'
---

# Extensibility

Pydantic AI is designed to be extended. [Capabilities](/docs/ai/core-concepts/capabilities) are the primary extension point — they bundle tools, lifecycle hooks, instructions, and model settings into reusable units that can be shared across agents, packaged as libraries, and loaded from [spec files](/docs/ai/core-concepts/agent-spec).

Beyond capabilities, Pydantic AI provides several other extension mechanisms for specialized needs.

Capabilities are the recommended way to extend Pydantic AI. They are useful for:

- **Teams**building reusable internal agent components (guardrails, audit logging, authentication)
- **Package authors**shipping extensions that work across models and agents
- **Community contributors**sharing solutions to common problems

See [Capabilities](/docs/ai/core-concepts/capabilities) for using and building capabilities, and [Hooks](/docs/ai/core-concepts/hooks) for the lightweight decorator-based approach.

To make a capability installable and usable in [agent specs](/docs/ai/core-concepts/agent-spec):

- 
**Implement**— defaults to the class name. Return`get_serialization_name()``None`to opt out of spec support.
- 
**Implement**— defaults to`from_spec()``cls(*args, **kwargs)`. Override when your constructor takes non-serializable types.
- 
**Package naming**— use the`pydantic-ai-`prefix (e.g.`pydantic-ai-guardrails`) so users can find your package.
- 
**Registration**— users pass custom capability types via`custom_capability_types`on`Agent.from_spec``Agent.from_file`

```
from pydantic_ai import Agent
from my_package import MyCapability
agent = Agent.from_file('agent.yaml', custom_capability_types=[MyCapability])
```
See [Custom capabilities in specs](/docs/ai/core-concepts/agent-spec#custom-capabilities-in-specs) for implementation details.

[ Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) is the official capability library for Pydantic AI — standalone capabilities like memory, guardrails, and context management live there rather than in core. See 

[What goes where?](https://pydantic.dev/docs/ai/harness/#what-goes-where)for the full breakdown, or jump to the

[capability matrix](https://github.com/pydantic/pydantic-ai-harness#capability-matrix).

[Capabilities](/docs/ai/core-concepts/capabilities) are the recommended extension mechanism for packages that need to bundle tools with hooks, instructions, or model settings. See [Third-party capabilities](/docs/ai/core-concepts/capabilities#third-party-capabilities) for community packages.

Many third-party extensions are available as [toolsets](/docs/ai/tools-toolsets/toolsets), which can also be wrapped as [capabilities](/docs/ai/core-concepts/capabilities) to take advantage of hooks, instructions, and model settings. See [Third-party toolsets](/docs/ai/tools-toolsets/toolsets#third-party-toolsets) for the full list.

For specialized tool execution needs (custom transport, tool filtering, execution wrapping), implement [ AbstractToolset](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.AbstractToolset) or subclass 

[:](/docs/ai/api/pydantic-ai/toolsets/#pydantic_ai.toolsets.WrapperToolset)

`WrapperToolset`- `AbstractToolset`
- `WrapperToolset`

See [Building a Custom Toolset](/docs/ai/tools-toolsets/toolsets#building-a-custom-toolset) for details.

For connecting to model providers not yet supported by Pydantic AI, implement [ Model](/docs/ai/api/models/base/#pydantic_ai.models.Model):

- `Model`
- `WrapperModel`

See [Custom Models](/docs/ai/models/overview#custom-models) for details.

For custom agent behavior, subclass [ AbstractAgent](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.AbstractAgent) or 

[:](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.WrapperAgent)

`WrapperAgent`- `AbstractAgent`- `run`,- `run_sync`, and- `run_stream`
- `WrapperAgent`

# Citations

1. Source page: https://pydantic.dev/docs/ai/guides/extensibility
