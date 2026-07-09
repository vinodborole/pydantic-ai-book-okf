---
type: Web Page
title: Dependencies | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/dependencies
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Dependencies

Pydantic AI uses a dependency injection system to provide data and services to your agent’s [system prompts](/docs/ai/core-concepts/agent#system-prompts), [tools](/docs/ai/tools-toolsets/tools) and [output validators](/docs/ai/core-concepts/output#output-validator-functions).

Matching Pydantic AI’s design philosophy, our dependency system tries to use existing best practice in Python development rather than inventing esoteric “magic”, this should make dependencies type-safe, understandable, easier to test, and ultimately easier to deploy in production.

Dependencies can be any python type. While in simple cases you might be able to pass a single object as a dependency (e.g. an HTTP connection), [dataclasses](https://docs.python.org/3/library/dataclasses.html#module-dataclasses) are generally a convenient container when your dependencies included multiple objects.

Here’s an example of defining an agent that requires dependencies.

(**Note:** dependencies aren’t actually used in this example, see [Accessing Dependencies](#accessing-dependencies) below)

Define a dataclass to hold dependencies.

Pass the dataclass type to the `deps_type` argument of the [ Agent constructor](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__). 

**Note**: we're passing the type here, NOT an instance, this parameter is not actually used at runtime, it's here so we can get full type checking of the agent.

When running the agent, pass an instance of the dataclass to the `deps` parameter.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Dependencies are accessed through the [ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) type, this should be the first parameter of system prompt functions etc.

[ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) may optionally be passed to a 

[function as the only argument.](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.system_prompt)

`system_prompt`[ RunContext](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) is parameterized with the type of the dependencies, if this type is incorrect, static type checkers will raise an error.

Access dependencies through the [ .deps](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.deps) attribute.

Access dependencies through the [ .deps](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.deps) attribute.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

In addition to [ .deps](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.deps), 

[provides access to the running agent via](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext)

`RunContext`[, which is useful when](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext.agent)

`.agent`[tools](/docs/ai/tools-toolsets/tools),

[hooks](/docs/ai/core-concepts/hooks), or

[capabilities](/docs/ai/core-concepts/capabilities)need to read agent properties like

[or](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.name)

`name`[.](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.output_type)

`output_type`Dependency fields can also be referenced in instructions and descriptions via [template strings](/docs/ai/core-concepts/agent-spec#template-strings) — for example, `TemplateStr('Hello {{name}}')` renders `name` from the deps object at runtime. This is especially useful in [agent specs](/docs/ai/core-concepts/agent-spec) where callables aren’t available.

[System prompt functions](/docs/ai/core-concepts/agent#system-prompts), [function tools](/docs/ai/tools-toolsets/tools) and [output validators](/docs/ai/core-concepts/output#output-validator-functions) are all run in the async context of an agent run.

If these functions are not coroutines (e.g. `async def`) they are called with
[ run_in_executor](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.run_in_executor) in a thread pool. It’s therefore marginally preferable
to use 

`async` methods where dependencies perform IO, although synchronous dependencies should work fine too.Here’s the same example as above, but with a synchronous dependency:

Here we use a synchronous `httpx.Client` instead of an asynchronous `httpx.AsyncClient`.

To match the synchronous dependency, the system prompt function is now a plain function, not a coroutine.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

As well as system prompts, dependencies can be used in [tools](/docs/ai/tools-toolsets/tools) and [output validators](/docs/ai/core-concepts/output#output-validator-functions).

To pass `RunContext` to a tool, use the [ tool](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.tool) decorator.

`RunContext` may optionally be passed to a [ output_validator](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.output_validator) function as the first argument.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

When testing agents, it’s useful to be able to customise dependencies.

While this can sometimes be done by calling the agent directly within unit tests, we can also override dependencies while calling application code which in turn calls the agent.

This is done via the [ override](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.override) method on the agent.

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
