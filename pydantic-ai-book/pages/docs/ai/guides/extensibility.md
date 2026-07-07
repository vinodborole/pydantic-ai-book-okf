---
type: Web Page
title: Extensibility | Pydantic Docs
resource: https://pydantic.dev/docs/ai/guides/extensibility
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Extensibility

Pydantic AI is designed to be extended. Capabilities are the primary extension point — they bundle tools, lifecycle hooks, instructions, and model settings into reusable units that can be shared across agents, packaged as libraries, and loaded from spec files.

Beyond capabilities, Pydantic AI provides several other extension mechanisms for specialized needs.

Capabilities are the recommended way to extend Pydantic AI. They are useful for:

- **Teams**building reusable internal agent components (guardrails, audit logging, authentication)
- **Package authors**shipping extensions that work across models and agents
- **Community contributors**sharing solutions to common problems

See Capabilities for using and building capabilities, and Hooks for the lightweight decorator-based approach.

To make a capability installable and usable in agent specs:

- 
**Implement**— defaults to the class name. Return`get_serialization_name()``None`to opt out of spec support.
- 
**Implement**— defaults to`from_spec()``cls(*args, **kwargs)`. Override when your constructor takes non-serializable types.
- 
**Package naming**— use the`pydantic-ai-`prefix (e.g.`pydantic-ai-guardrails`) so users can find your package.
- 
**Registration**— users pass custom capability types via`custom_capability_types`on`Agent.from_spec`or`Agent.from_file`.

```
from pydantic_ai import Agent
from my_package import MyCapability
agent = Agent.from_file('agent.yaml', custom_capability_types=[MyCapability])
```
See Custom capabilities in specs for implementation details.

**Pydantic AI Harness** is the official capability library for Pydantic AI — standalone capabilities like memory, guardrails, and context management live there rather than in core. See What goes where? for the full breakdown, or jump to the capability matrix.

Capabilities are the recommended extension mechanism for packages that need to bundle tools with hooks, instructions, or model settings. See Third-party capabilities for community packages.

Many third-party extensions are available as toolsets, which can also be wrapped as capabilities to take advantage of hooks, instructions, and model settings. See Third-party toolsets for the full list.

For specialized tool execution needs (custom transport, tool filtering, execution wrapping), implement `AbstractToolset` or subclass `WrapperToolset`:

- `AbstractToolset`— full control over tool definitions and execution
- `WrapperToolset`— delegates to a wrapped toolset, override specific methods

See Building a Custom Toolset for details.

For connecting to model providers not yet supported by Pydantic AI, implement `Model`:

- `Model`— the base interface for model implementations
- `WrapperModel`— delegates to a wrapped model, useful for adding instrumentation or transformations

See Custom Models for details.

For custom agent behavior, subclass `AbstractAgent` or `WrapperAgent`:

- `AbstractAgent`— the base interface for agent implementations, providing- `run`,- `run_sync`, and- `run_stream`
- `WrapperAgent`— delegates to a wrapped agent, useful for adding pre/post-processing or context management

# Citations

1. Source page: https://pydantic.dev/docs/ai/guides/extensibility
