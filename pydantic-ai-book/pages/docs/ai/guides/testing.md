---
type: Web Page
title: Testing | Pydantic Docs
resource: https://pydantic.dev/docs/ai/guides/testing
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Testing

Writing unit tests for Pydantic AI code is just like unit tests for any other Python code.

Because for the most part they’re nothing new, we have pretty well established tools and patterns for writing and running these kinds of tests.

Unless you’re really sure you know better, you’ll probably want to follow roughly this strategy:

- Use `pytest`
- If you find yourself typing out long assertions, use [inline-snapshot](https://15r10nk.github.io/inline-snapshot/latest/)
- Similarly, [dirty-equals](https://dirty-equals.helpmanual.io/latest/)can be useful for comparing large data structures
- Use `TestModel``FunctionModel`
- Use `Agent.override`
- Set `ALLOW_MODEL_REQUESTS=False`

The simplest and fastest way to exercise most of your application code is using [ TestModel](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel), this will (by default) call all tools in the agent, then return either plain text or a structured response depending on the return type of the agent.

Let’s write unit tests for the following application code:

`DatabaseConn` is a class that holds a database connection

`WeatherService` has methods to get weather forecasts and historic data about the weather

We need to call a different endpoint depending on whether the date is in the past or the future, you'll see why this nuance is important below

This function is the code we want to test, together with the agent it uses

Here we have a function that takes a list of `(user_prompt, user_id)` tuples, gets a weather forecast for each prompt, and stores the result in the database.

**We want to test this code without having to mock certain objects or modify our code so we can pass test objects in.**

Here’s how we would write tests using [ TestModel](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel):

We're using [anyio](https://anyio.readthedocs.io/en/stable/) to run async tests.

This is a safety measure to make sure we don't accidentally make real requests to the LLM while testing, see [ ALLOW_MODEL_REQUESTS](/docs/ai/api/models/base/#pydantic_ai.models.ALLOW_MODEL_REQUESTS) for more details.

We're using [ Agent.override](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.override) to replace the agent's model with 

[, the nice thing about](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel)

`TestModel``override` is that we can replace the model inside agent without needing access to the agent `run*` methods call site.Now we call the function we want to test inside the `override` context manager.

But default, `TestModel` will return a JSON string summarising the tools calls made, and what was returned. If you wanted to customise the response to something more closely aligned with the domain, you could add [ custom_output_text='Sunny'](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel.custom_output_text) when defining 

`TestModel`.So far we don't actually know which tools were called and with which values, we can use [ capture_run_messages](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.capture_run_messages) to inspect messages from the most recent run and assert the exchange between the agent and the model occurred as expected.

The [ IsNow](https://dirty-equals.helpmanual.io/latest/types/datetime/#dirty_equals.IsNow) helper allows us to use declarative asserts even with data which will contain timestamps that change over time.

`TestModel` isn't doing anything clever to extract values from the prompt, so these values are hardcoded.

The above tests are a great start, but careful readers will notice that the `WeatherService.get_forecast` is never called since `TestModel` calls `weather_forecast` with a date in the past.

To fully exercise `weather_forecast`, we need to use [ FunctionModel](/docs/ai/api/models/function/#pydantic_ai.models.function.FunctionModel) to customise how the tools is called.

Here’s an example of using `FunctionModel` to test the `weather_forecast` tool with custom inputs

We define a function `call_weather_forecast` that will be called by `FunctionModel` in place of the LLM, this function has access to the list of [ ModelMessage](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelMessage)s that make up the run, and 

[which contains information about the agent and the function tools and return tools.](/docs/ai/api/models/function/#pydantic_ai.models.function.AgentInfo)

`AgentInfo`Our function is slightly intelligent in that it tries to extract a date from the prompt, but just hard codes the location.

We use [ FunctionModel](/docs/ai/api/models/function/#pydantic_ai.models.function.FunctionModel) to replace the agent's model with our custom function.

If you’re writing lots of tests that all require model to be overridden, you can use [pytest fixtures](https://docs.pytest.org/en/6.2.x/fixture.html) to override the model with [ TestModel](/docs/ai/api/models/test/#pydantic_ai.models.test.TestModel) or 

[in a reusable way.](/docs/ai/api/models/function/#pydantic_ai.models.function.FunctionModel)

`FunctionModel`Here’s an example of a fixture that overrides the model with `TestModel`:

# Citations

1. Source page: https://pydantic.dev/docs/ai/guides/testing
