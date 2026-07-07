---
type: Web Page
title: Dependencies | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/dependencies
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Dependencies

Pydantic AI uses a dependency injection system to provide data and services to your agent’s system prompts, tools and output validators.

Matching Pydantic AI’s design philosophy, our dependency system tries to use existing best practice in Python development rather than inventing esoteric “magic”, this should make dependencies type-safe, understandable, easier to test, and ultimately easier to deploy in production.

Dependencies can be any python type. While in simple cases you might be able to pass a single object as a dependency (e.g. an HTTP connection), dataclasses are generally a convenient container when your dependencies included multiple objects.

Here’s an example of defining an agent that requires dependencies.

(**Note:** dependencies aren’t actually used in this example, see Accessing Dependencies below)

Define a dataclass to hold dependencies.

Pass the dataclass type to the `deps_type` argument of the `Agent` constructor. **Note**: we're passing the type here, NOT an instance, this parameter is not actually used at runtime, it's here so we can get full type checking of the agent.

When running the agent, pass an instance of the dataclass to the `deps` parameter.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Dependencies are accessed through the `RunContext` type, this should be the first parameter of system prompt functions etc.

`RunContext` may optionally be passed to a `system_prompt` function as the only argument.

`RunContext` is parameterized with the type of the dependencies, if this type is incorrect, static type checkers will raise an error.

Access dependencies through the `.deps` attribute.

Access dependencies through the `.deps` attribute.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

In addition to `.deps`, `RunContext` provides access to the running agent via `.agent`, which is useful when tools, hooks, or capabilities need to read agent properties like `name` or `output_type`.

Dependency fields can also be referenced in instructions and descriptions via template strings — for example, `TemplateStr('Hello {{name}}')` renders `name` from the deps object at runtime. This is especially useful in agent specs where callables aren’t available.

System prompt functions, function tools and output validators are all run in the async context of an agent run.

If these functions are not coroutines (e.g. `async def`) they are called with
`run_in_executor` in a thread pool. It’s therefore marginally preferable
to use `async` methods where dependencies perform IO, although synchronous dependencies should work fine too.

Here’s the same example as above, but with a synchronous dependency:

Here we use a synchronous `httpx.Client` instead of an asynchronous `httpx.AsyncClient`.

To match the synchronous dependency, the system prompt function is now a plain function, not a coroutine.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

As well as system prompts, dependencies can be used in tools and output validators.

To pass `RunContext` to a tool, use the `tool` decorator.

`RunContext` may optionally be passed to a `output_validator` function as the first argument.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

When testing agents, it’s useful to be able to customise dependencies.

While this can sometimes be done by calling the agent directly within unit tests, we can also override dependencies while calling application code which in turn calls the agent.

This is done via the `override` method on the agent.

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
