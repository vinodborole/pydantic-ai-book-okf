---
type: Web Page
title: Output | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/output
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Output

“Output” refers to the final value returned from running an agent. This can be either plain text, structured data, an image, or the result of a function called with arguments provided by the model.

The output is wrapped in `AgentRunResult` or `StreamedRunResult` so that you can access other data, like usage of the run and message history.

Both `AgentRunResult` and `StreamedRunResult` are generic in the data they wrap, so typing information about the data returned by the agent is preserved.

A run ends when the model responds with one of the output types, or, if no output type is specified or `str` is one of the allowed options, when a plain text response is received. A run can also be cancelled if usage limits are exceeded, see Usage Limits.

Here’s an example using a Pydantic model as the `output_type`, forcing the model to respond with data matching our specification:

*(This example is complete, it can be run “as is”)*

The `Agent` class constructor takes an `output_type` argument that takes one or more types or output functions. It supports simple scalar types, list and dict types (including `TypedDict`s and `StructuredDict`s), dataclasses and Pydantic models, as well as type unions — generally everything supported as type hints in a Pydantic model. You can also pass a list of multiple choices.

By default, Pydantic AI leverages the model’s tool calling capability to make it return structured data. When multiple output types are specified (in a union or list), each member is registered with the model as a separate output tool in order to reduce the complexity of the schema and maximise the chances a model will respond correctly. This has been shown to work well across a wide range of models. If you’d like to change the names of the output tools, use a model’s native structured output feature, or pass the output schema to the model in its instructions, you can use an output mode marker class.

When no output type is specified, or when `str` is among the output types, any plain text response from the model will be used as the output data.
If `str` is not among the output types, the model is forced to return structured data or call an output function.

If the output type schema is not of type `"object"` (e.g. it’s `int` or `list[int]`), the output type is wrapped in a single element object, so the schema of all tools registered with the model are object schemas.

Structured outputs (like tools) use Pydantic to build the JSON schema used for the tool, and to validate the data returned by the model.

Here’s an example of returning either text or structured data:

This could also have been a union: `output_type=Box | str`. However, as explained in the "Type checking considerations" section above, that would've required explicitly specifying the generic parameters on the `Agent` constructor and adding `# type: ignore` to this line in order to be type checked correctly.

*(This example is complete, it can be run “as is”)*

Here’s an example of using a union return type, which will register multiple output tools and wrap non-object schemas in an object:

As explained in the "Type checking considerations" section above, using a union rather than a list requires explicitly specifying the generic parameters on the `Agent` constructor and adding `# type: ignore` to this line in order to be type checked correctly.

*(This example is complete, it can be run “as is”)*

Instead of plain text or structured data, you may want the output of your agent run to be the result of a function called with arguments provided by the model, for example to further process or validate the data provided through the arguments (with the option to tell the model to try again), or to hand off to another agent.

Output functions are similar to function tools, but the model is forced to call one of them, the call ends the agent run, and the result is not passed back to the model.

As with tool functions, output function arguments provided by the model are validated using Pydantic (with optional validation context), can optionally take `RunContext` as the first argument, and can raise `ModelRetry` to ask the model to try again with modified arguments (or with a different output type).

To specify output functions, you set the agent’s `output_type` to either a single function (or bound instance method), or a list of functions. The list can also contain other output types like simple scalars or entire Pydantic models.
You typically do not want to also register your output function as a tool (using the `@agent.tool` decorator or `tools` argument), as this could confuse the model about which it should be calling.

Here’s an example of all of these features in action:

If you provide an output function that takes a string, Pydantic AI will by default create an output tool like for any other output function. If instead you’d like the model to provide the string using plain text output, you can wrap the function in the `TextOutput` marker class.

If desired, this marker class can be used alongside one or more `ToolOutput` marker classes (or unmarked types or functions) in a list provided to `output_type`.

Like other output functions, text output functions can optionally take `RunContext` as the first argument, and can raise `ModelRetry` to ask the model to try again with modified arguments (or with a different output type).

*(This example is complete, it can be run “as is”)*

When streaming with `run_stream()` or `run_stream_sync()`, output functions are called **multiple times** — once for each partial output received from the model, and once for the final complete output.

You should check the `RunContext.partial_output` flag when your output function has **side effects** (e.g., sending notifications, logging, database updates) that should only execute on the final output.

When streaming, `partial_output` is `True` for each partial output and `False` for the final complete output.
For all other run methods, `partial_output` is always `False` as the function is only called once with the complete output.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Pydantic AI implements three different methods to get a model to output structured data:

- Tool Output, where tool calls are used to produce the output.
- Native Output, where the model is required to produce text content compliant with a provided JSON schema.
- Prompted Output, where a prompt is injected into the model instructions including the desired JSON schema, and we attempt to parse the model’s plain-text response as appropriate.

In the default Tool Output mode, the output JSON schema of each output type (or function) is provided to the model as the parameters schema of a special output tool. This is the default as it’s supported by virtually all models and has been shown to work very well.

If you’d like to change the name of the output tool, pass a custom description to aid the model, or turn on or off strict mode, you can wrap the type(s) in the `ToolOutput` marker class and provide the appropriate arguments. Note that by default, the description is taken from the docstring specified on a Pydantic model or output function, so specifying it using the marker class is typically not necessary.

When using output tools, each tool gets its own retry counter — the output side of the agent retry budget (set with `AgentRetries` via `Agent(retries={'output': N})`, or per-run via `agent.run(retries={'output': N})`) is the *default per-tool limit*. To override the limit for an individual output tool, pass `max_retries` on `ToolOutput`: `ToolOutput(Fruit, max_retries=2)`. See How output retries are enforced for the relationship to the text-output path’s global budget.

To dynamically modify or filter the available output tools during an agent run, you can define an agent-wide `prepare_output_tools` function that will be called ahead of each step of a run. This function should be of type `ToolsPrepareFunc`, which takes the `RunContext` and a list of `ToolDefinition`, and returns the output tool definitions to expose for that step. Return `[]` to expose no output tools. This is analogous to the `prepare_tools` function for non-output tools.

If we were passing just `Fruit` and `Vehicle` without custom tool names, we could have used a union: `output_type=Fruit | Vehicle`. However, as `ToolOutput` is an object rather than a type, we have to use a list.

*(This example is complete, it can be run “as is”)*

An output tool call is what ends a run and produces its final result. When a model emits one in the *same* response as other tool calls, the agent’s `end_strategy` decides what happens to the rest. Most agents never need to think about this, since most responses don’t mix an output tool with other tools — but when one does, `end_strategy` controls how those calls run and which one becomes the final result.

| Strategy | Output tools | Function tools — output succeeded | Function tools — every output failed | 
|---|---|---|---|
| `'graceful'`(default) | Run in emission order; first success is the final result, later output tools skipped | Run, in parallel where possible, in emission order | Run; the run continues | 
| `'early'` | Run in emission order; the run ends at the first success | Skipped | Run; the run continues | 
| `'exhaustive'` | All run, in parallel; first valid result by emission order wins | Run, in parallel | Run; the run continues | 

`'graceful'` is the default and the right choice for most agents: function tools the model requested alongside an output tool still run, so their side effects happen and their results are available to the model if the run continues. Only the first successful output tool is used; later output tools are skipped so their side effects don’t fire more than once.

Choose `'early'` to end the run the instant an output tool succeeds — function tools requested in the same response are then skipped entirely. This is the fastest option when you never need those function tools to run once you have a result.

Choose `'exhaustive'` to run every tool, including additional output tools whose results won’t be used. This gives the model full visibility that each tool ran, at the cost of executing output-tool side effects that are ultimately discarded.

When *every* output tool fails, function tools run and the run continues under all three strategies: there is no result to end on, so the output failures go back to the model as retries and the function tools the model also asked for are run, letting it react to both on the next round.

Under the `'graceful'` and `'exhaustive'` end strategies, function tools requested alongside an output tool still run. If one of them raises `ModelRetry` (or its arguments fail validation) in the same response as a successful output tool, the output result is **not** used as the final result. Instead, the retry is sent back to the model so it can correct the problem, since the output may have been based on the failed tool call. This does not apply under `'early'`, where function tools don’t run once an output succeeds, nor when streaming, where the first matching output is committed immediately.

Like function tools, output tools run concurrently. Under the `'exhaustive'` end strategy, where multiple output tools can run in parallel, you can make an output tool a barrier with `ToolOutput(sequential=True)` — useful when you want all of a response’s function tools to finish before the output tool runs. This is the output-tool counterpart of the `sequential=True` flag for function tools; see Parallel tool calls & concurrency for how the barrier behaves and how to run an entire run’s tools serially.

Native Output mode uses a model’s native “Structured Outputs” feature (aka “JSON Schema response format”), where the model is forced to only output text matching the provided JSON schema. Note that this is not supported by all models, and sometimes comes with restrictions. For example, Gemini cannot use tools at the same time as structured output, and attempting to do so will result in an error.

To use this mode, you can wrap the output type(s) in the `NativeOutput` marker class that also lets you specify a `name` and `description` if the name and docstring of the type or function are not sufficient.

This could also have been a union: `output_type=Fruit | Vehicle`. However, as explained in the "Type checking considerations" section above, that would've required explicitly specifying the generic parameters on the `Agent` constructor and adding `# type: ignore` to this line in order to be type checked correctly.

*(This example is complete, it can be run “as is”)*

In this mode, the model is prompted to output text matching the provided JSON schema through its instructions and it’s up to the model to interpret those instructions correctly. This is usable with all models, but is often the least reliable approach as the model is not forced to match the schema.

While we would generally suggest starting with tool or native output, in some cases this mode may result in higher quality outputs, and for models without native tool calling or structured output support it is the only option for producing structured outputs.

If the model API supports the “JSON Mode” feature (aka “JSON Object response format”) to force the model to output valid JSON, this is enabled, but it’s still up to the model to abide by the schema. Pydantic AI will validate the returned structured data and tell the model to try again if validation fails, but if the model is not intelligent enough this may not be sufficient.

To use this mode, you can wrap the output type(s) in the `PromptedOutput` marker class that also lets you specify a `name` and `description` if the name and docstring of the type or function are not sufficient. Additionally, `template` lets you specify a custom instructions template to be used instead of the default, or `template=False` to disable the schema prompt entirely.

This could also have been a union: `output_type=Vehicle | Device`. However, as explained in the "Type checking considerations" section above, that would've required explicitly specifying the generic parameters on the `Agent` constructor and adding `# type: ignore` to this line in order to be type checked correctly.

*(This example is complete, it can be run “as is”)*

If it’s not feasible to define your desired structured output object using a Pydantic `BaseModel`, dataclass, or `TypedDict`, for example when you get a JSON schema from an external source or generate it dynamically, you can use the `StructuredDict()` helper function to generate a `dict[str, Any]` subclass with a JSON schema attached that Pydantic AI will pass to the model.

Note that Pydantic AI will not perform any validation of the received JSON object and it’s up to the model to correctly interpret the schema and any constraints expressed in it, like required fields or integer value ranges.

The output type will be a `dict[str, Any]` and it’s up to your code to defensively read from it in case the model made a mistake. You can use an output validator to reflect validation errors back to the model and get it to try again.

Along with the JSON schema, you can optionally pass `name` and `description` arguments to provide additional context to the model:

```
from pydantic_ai import Agent, StructuredDict
HumanDict = StructuredDict(
    {
        'type': 'object',
        'properties': {
            'name': {'type': 'string'},
            'age': {'type': 'integer'}
        },
        'required': ['name', 'age']
    },
    name='Human',
    description='A human with a name and age',
)
agent = Agent('openai:gpt-5.2', output_type=HumanDict)
result = agent.run_sync('Create a person')
#> {'name': 'John Doe', 'age': 30}
```
Some validation relies on an extra Pydantic context object. You can pass such an object to an `Agent` at definition-time via its `validation_context` parameter. It will be used in the validation of both structured outputs and tool arguments.

This validation context can be either:

- the context object itself (`Any`), used as-is to validate outputs, or
- a function that takes the `RunContext`and returns a context object (`Any`). This function will be called automatically before each validation, allowing you to build a dynamic validation context.

*(This example is complete, it can be run “as is”)*

Some validation is inconvenient or impossible to do in Pydantic validators, in particular when the validation requires IO and is asynchronous. Pydantic AI provides a way to add validation functions via the `agent.output_validator` decorator.

Each `ModelRetry` raised here consumes one unit of the run’s output retry budget. The budget defaults to `1` and can be set on the agent with `AgentRetries` via `Agent(retries={'output': N})`, on a single run via `agent.run(retries={'output': N})`, or per output tool via `ToolOutput(max_retries=N)`. Inside the validator, `ctx.max_retries` reflects the limit that will actually stop you (the global budget on the text path, or the per-tool limit on the tool path) and `ctx.retry` is the global retry counter, so it stays consistent across output-tool switches within a single run. See How output retries are enforced for the full enforcement model.

If you want to implement separate validation logic for different output types, it’s recommended to use output functions instead, to save you from having to do `isinstance` checks inside the output validator.
If you want the model to output plain text, do your own processing or validation, and then have the agent’s final output be the result of your function, it’s recommended to use an output function with the `TextOutput` marker class.

Here’s a simplified variant of the SQL Generation example:

*(This example is complete, it can be run “as is”)*

When streaming with `run_stream()` or `run_stream_sync()`, output validators are called **multiple times** — once for each partial output received from the model, and once for the final complete output.

You should check the `RunContext.partial_output` flag when you want to **validate only the complete result**, not intermediate partial values.

When streaming, `partial_output` is `True` for each partial output and `False` for the final complete output.
For all other run methods, `partial_output` is always `False` as the validator is only called once with the complete output.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Some models can generate images as part of their response, for example those that support the Image Generation native tool and OpenAI models using the Code Execution native tool when told to generate a chart.

To use the generated image as the output of the agent run, you can set `output_type` to `BinaryImage`. If no image-generating native tool is explicitly specified, the `ImageGenerationTool` will be enabled automatically.

*(This example is complete, it can be run “as is”)*

If an agent does not need to always generate an image, you can use a union of `BinaryImage` and `str`. If the model generates both, the image will take precedence as output and the text will be available on `ModelResponse.text`:

Some agents perform their work entirely through tool calls and don’t need to produce a final output — for example, an agent that updates a record via a tool and then stops. Certain models (notably Anthropic) will return an empty response in this case, which by default causes Pydantic AI to retry until the model produces content.

To instead treat an empty response as a successful run, include `None` in the `output_type`:

When the model returns an empty response and `None` is an allowed output type, the agent will return `None` instead of retrying. Output validator functions still run with `None` as the argument, so you can raise `ModelRetry` to reject it if needed.

`output_type=str | None` is the canonical case: it’s handled as regular text output, and the **only** way the model signals `None` is by returning a response with no text output — either an empty response, or one containing only thinking content, which some reasoning models emit after completing their work through a tool call. There’s no output tool or structured schema involved. This mirrors how plain `str` is already treated specially as free-form text output rather than a structured tool call.

`None` is also supported in the other output modes, with an extra structured commit path in addition to (or in place of) the empty-response fallback:

- **Bare unions including**— e.g.- `None`that use tool mode- `output_type=int | None`,- `output_type=[int, float, None]`, or- `output_type=[ToolOutput(Foo), None]`: a dedicated- `final_result_NoneType`output tool is exposed alongside the other output tools, so the model can commit to- `None`through a tool call. An empty or thinking-only model response is still also treated as- `None`, as with- `str | None`.
- **Explicit output mode markers**— e.g.- `output_type=ToolOutput(int | None)`,- `output_type=NativeOutput([int, None])`, or- `output_type=PromptedOutput([int, None])`:- `None`is included as a branch of the structured schema the wrapper generates. The model commits by calling the tool with- `null`(for- `ToolOutput`) or by selecting the- `NoneType`branch of the discriminated schema (for- `NativeOutput`/- `PromptedOutput`). An empty response is- **not**accepted — once you’ve opted into an explicit structured output mode, the model is expected to commit through the schema.

There two main challenges with streamed results:

- Validating structured responses before they’re complete, this is achieved by “partial validation” which was recently added to Pydantic in pydantic/pydantic#10748.
- When receiving a response, we don’t know if it’s the final response without starting to stream it and peeking at the content. Pydantic AI streams just enough of the response to sniff out if it’s a tool call or an output, then streams the whole thing and calls tools, or returns the stream as a `StreamedRunResult`.

Example of streamed text output:

Streaming works with the standard `Agent` class, and doesn't require any special setup, just a model that supports streaming (currently all models support streaming).

The `Agent.run_stream()` method is used to start a streamed run, this method returns a context manager so the connection can be closed when the stream completes.

Each item yield by `StreamedRunResult.stream_text()` is the complete text response, extended as new data is received.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

The optional `debounce_by` argument of `stream_text()` controls how long Pydantic AI groups incoming chunks before yielding. The default `0.1` groups chunks for up to 0.1 seconds; pass `None` to yield as soon as each chunk arrives. Debouncing is especially helpful for long structured responses, where it reduces the overhead of validating each chunk as it arrives.

We can also stream text as deltas rather than the entire text in each item:

`stream_text` will error if the response is not text.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Here’s an example of streaming a user profile as it’s built:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

As setting an `output_type` uses the Tool Output mode by default, this will only work if the model supports streaming tool arguments. For models that don’t, like Gemini, try Native Output or Prompted Output instead.

If you want fine-grained control of validation, you can use the following pattern to get the entire partial `ModelResponse`:

`stream_response` streams the data as `ModelResponse` objects, thus iteration can't fail with a `ValidationError`.

`validate_response_output` validates the data, `allow_partial=True` enables pydantic's `experimental_allow_partial` flag on `TypeAdapter`.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Sometimes you need to stop a streaming response before it completes: a user clicks “stop generating” in a chat UI, you’ve received enough data to make a decision, or you want to avoid receiving more tokens. `run_stream()` and `iter()` support explicit cancellation by closing the underlying model stream. `run_stream_events()` is an async context manager, so cleanup runs deterministically when you stop consuming events — leaving the `async with` block cancels the background run task.

`run_stream_events()` is an async context manager that yields an async iterator over events:

Breaking out of the loop leaves the `async with` block, which cancels the background run task and closes the HTTP connection.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

`run_stream_events()` does not expose a `cancel()` method. If you need an explicit model-response cancellation handle, use `run_stream()` or `agent.iter()`.

Call `cancel()` on the `StreamedRunResult` to cancel the stream:

Check a condition during streaming, for example whether enough text has been received.

`cancel()` tells the model provider to stop generating tokens and closes the HTTP connection when the model integration supports it.

The `cancelled` property reflects the cancellation state.

The final `ModelResponse` is marked with `state='interrupted'` so that downstream code can identify incomplete responses.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

If you `break` out of `stream_text()` and then leave the surrounding `async with` block, the stream is cleaned up as the context exits. Use `cancel()` when you want to stop generation immediately instead of only stopping local consumption.

When using `agent.iter()` for fine-grained control over the agent graph, you can cancel the `AgentStream` inside a `ModelRequestNode.stream()` context:

`AgentStream.cancel()` cancels the stream at the model request level.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

When a stream is cancelled mid-generation, the response is recorded with `state='interrupted'` in the message history. The history includes any partial content that was received before cancellation:

The message history includes the interrupted response with any partial content that was received before cancellation.

The interrupted response state lets your application decide whether to keep, inspect, or discard the partial response before reusing the history.

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

The following examples demonstrate how to use streamed responses in Pydantic AI:

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/output
