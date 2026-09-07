---
type: Web Page
title: Dependencies | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/dependencies
timestamp: '2026-09-07T12:01:58.556264+00:00'
---

# Dependencies

Pydantic AI uses a dependency injection system to provide data and services to your agent’s [system prompts](/docs/ai/core-concepts/agent/#system-prompts), [tools](/docs/ai/tools-toolsets/tools/) and [output validators](/docs/ai/core-concepts/output/#output-validator-functions).

Pydantic AI’s dependency system follows established Python practices, making dependencies type-safe, understandable, easy to test, and easy to deploy in production.

Dependencies can be any python type. While in simple cases you might be able to pass a single object as a dependency (e.g. an HTTP connection), [dataclasses](https://docs.python.org/3/library/dataclasses.html#module-dataclasses) are generally a convenient container when your dependencies included multiple objects.

Here’s an example of defining an agent that requires dependencies.

(**Note:** dependencies aren’t actually used in this example, see [Accessing Dependencies](#accessing-dependencies) below)

Define a dataclass to hold dependencies.

Pass the dataclass type to the `deps_type` argument of the [`Agent` constructor](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__). **Note**: we're passing the type here, NOT an instance, this parameter is not actually used at runtime, it's here so we can get full type checking of the agent.

When running the agent, pass an instance of the dataclass to the `deps` parameter.

*(To run this example, ensure `asyncio` is imported and add `asyncio.run(main())`; no other changes are needed.)*

Dependencies are accessed through the [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) type, this should be the first parameter of system prompt functions etc.

[`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) may optionally be passed to a [`system_prompt`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.system_prompt) function as the only argument.

[`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) is parameterized with the type of the dependencies, if this type is incorrect, static type checkers will raise an error.

Access the HTTP client through the [`.deps`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.deps) attribute.

Access the API key through the same attribute.

*(To run this example, ensure `asyncio` is imported and add `asyncio.run(main())`; no other changes are needed.)*

In addition to [`.deps`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.deps), [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) provides access to the running agent via [`.agent`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.agent), which is useful when [tools](/docs/ai/tools-toolsets/tools/), [hooks](/docs/ai/core-concepts/hooks/), or [capabilities](/docs/ai/capabilities/overview/) need to read agent properties like [`name`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.name) or [`output_type`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.output_type). The [`.realtime`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.realtime) property identifies realtime sessions without requiring a model type check, and [`.realtime_session`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.realtime_session) exposes the live [`RealtimeSession`](/docs/ai/api/pydantic-ai/realtime/#pydantic_ai.realtime.RealtimeSession) to tools and hooks once it is connected.

Dependency fields can also be referenced in instructions and descriptions via [template strings](/docs/ai/core-concepts/agent-spec/#template-strings) — for example, [`TemplateStr('Hello {{name}}')`](/docs/ai/api/pydantic-ai/template/#pydantic_ai.template.TemplateStr) renders `name` from the deps object at runtime. This is especially useful in [agent specs](/docs/ai/core-concepts/agent-spec/) where callables aren’t available.

[System prompt functions](/docs/ai/core-concepts/agent/#system-prompts), [function tools](/docs/ai/tools-toolsets/tools/) and [output validators](/docs/ai/core-concepts/output/#output-validator-functions) are all run in the async context of an agent run.

If these functions are synchronous, defined with `def` rather than `async def`, Pydantic AI calls them with
[`run_in_executor`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.run_in_executor) in a thread pool. Prefer `async` functions when dependencies
perform I/O, although synchronous dependencies also work.

Here’s the same example as above, but with a synchronous dependency:

Here we use a synchronous `httpx.Client` instead of an asynchronous `httpx.AsyncClient`.

To match the synchronous dependency, the system prompt function is now a plain function, not a coroutine.

*(To run this example, ensure `asyncio` is imported and add `asyncio.run(main())`; no other changes are needed.)*

As well as system prompts, dependencies can be used in [tools](/docs/ai/tools-toolsets/tools/) and [output validators](/docs/ai/core-concepts/output/#output-validator-functions).

To pass `RunContext` to a tool, use the [`tool`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool) decorator.

`RunContext` may optionally be passed to a [`output_validator`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.output_validator) function as the first argument.

*(To run this example, ensure `asyncio` is imported and add `asyncio.run(main())`; no other changes are needed.)*

When testing agents, it’s useful to be able to customise dependencies.

While this can sometimes be done by calling the agent directly within unit tests, we can also override dependencies while calling application code which in turn calls the agent.

This is done via the [`override`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.override) method on the agent.

Define a method on the dependency to make the system prompt easier to customise.

Call the system prompt factory from within the system prompt function.

Application code that calls the agent, in a real application this might be an API endpoint.

Call the agent from within the application code, in a real application this call might be deep within a call stack. Note `app_deps` here will NOT be used when deps are overridden.

*(This example is complete, it can be run “as is”)*

Define a subclass of `MyDeps` in tests to customise the system prompt factory.

Create an instance of the test dependency, we don't need to pass an `http_client` here as it's not used.

Override the dependencies of the agent for the duration of the `with` block, `test_deps` will be used when the agent is run.

Now we can safely call our application code, the agent will use the overridden dependencies.

The following examples demonstrate how to use dependencies in Pydantic AI:

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/dependencies
