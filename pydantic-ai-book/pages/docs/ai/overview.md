---
type: Web Page
title: Pydantic AI | Pydantic Docs
resource: https://pydantic.dev/docs/ai/overview
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Pydantic AI

*GenAI Agent Framework, the Pydantic way*

Pydantic AI is a Python agent framework designed to help you quickly, confidently, and painlessly build production grade applications and workflows with Generative AI.

FastAPI revolutionized web development by offering an innovative and ergonomic design, built on the foundation of [Pydantic Validation](https://docs.pydantic.dev) and modern Python features like type hints.

Yet despite virtually every Python agent framework and LLM library using Pydantic Validation, when we began to use LLMs in [Pydantic Logfire](https://pydantic.dev/logfire), we couldn’t find anything that gave us the same feeling.

We built Pydantic AI with one simple aim: to bring that FastAPI feeling to GenAI app and agent development.

Pydantic AI ships the agent loop, a composable [capabilities](/docs/ai/capabilities/overview) system, and [built-in capabilities](/docs/ai/capabilities/overview#built-in-capabilities) for [thinking](/docs/ai/capabilities/thinking), [web search](/docs/ai/capabilities/web-search), [web fetch](/docs/ai/capabilities/web-fetch), [image generation](/docs/ai/capabilities/image-generation), [MCP](/docs/ai/capabilities/mcp), [tool search](/docs/ai/capabilities/tool-search), and more; [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) is our official library of ready-made capabilities — code execution, file access, guardrails, sub-agent orchestration, and more — that you pick and choose to build coding agents, research assistants, and anything in between.

1. 
**Built by the Pydantic Team** :[Pydantic Validation](https://docs.pydantic.dev/latest/) is the validation layer of the OpenAI SDK, the Google ADK, the Anthropic SDK, LangChain, LlamaIndex, AutoGPT, Transformers, CrewAI, Instructor and many more.*Why use the derivative when you can go straight to the source?* 😃
2. 
**Model-agnostic** :
Supports virtually every[model](/docs/ai/models/overview) and provider: OpenAI, Anthropic, Gemini, DeepSeek, Grok, Cohere, Mistral, and Perplexity; Azure AI Foundry, Amazon Bedrock, Google Cloud, Ollama, LiteLLM, Groq, OpenRouter, Together AI, Fireworks AI, Cerebras, Hugging Face, GitHub, Heroku, Vercel, Nebius, OVHcloud, Alibaba Cloud, SambaNova, and Z.AI. If your favorite model or provider is not listed, you can easily implement a[custom model](/docs/ai/models/overview#custom-models) .
3. 
**Seamless Observability** :
Tightly[integrates](/docs/ai/integrations/logfire) with[Pydantic Logfire](https://pydantic.dev/logfire) , our general-purpose OpenTelemetry observability platform, for real-time debugging, evals-based performance monitoring, and behavior, tracing, and cost tracking. If you already have an observability platform that supports OTel, you can[use that too](/docs/ai/integrations/logfire#alternative-observability-backends) .
4. 
**Fully Type-safe** :
Designed to give your IDE or AI coding agent as much context as possible for auto-completion and[type checking](/docs/ai/core-concepts/agent#static-type-checking) , moving entire classes of errors from runtime to write-time for a bit of that Rust “if it compiles, it works” feel.
5. 
**Powerful Evals** :
Enables you to systematically test and[evaluate](/docs/ai/evals/evals) the performance and accuracy of the agentic systems you build, and monitor the performance over time in Pydantic Logfire.
6. 
**Extensible by Design** :
Build agents from composable[capabilities](/docs/ai/capabilities/overview) that bundle tools, hooks, instructions, and model settings into reusable units. Use built-in capabilities for[web search](/docs/ai/capabilities/web-search) ,[thinking](/docs/ai/capabilities/thinking) , and[MCP](/docs/ai/capabilities/mcp) , pick from the[Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) capability library, build your own, or install[third-party capability packages](/docs/ai/guides/extensibility) . Define agents entirely in[YAML/JSON](/docs/ai/core-concepts/agent-spec) — no code required.
7. 
**MCP and UI** :
Integrates the[Model Context Protocol](/docs/ai/mcp/overview) and various[UI event stream](/docs/ai/integrations/ui/overview) standards to give your agent access to external tools and data and build interactive applications with streaming event-based communication.
8. 
**Human-in-the-Loop Tool Approval** :
Easily lets you flag that certain tool calls[require approval](/docs/ai/tools-toolsets/deferred-tools#human-in-the-loop-tool-approval) before they can proceed, possibly depending on tool call arguments, conversation history, or user preferences.
9. 
**Durable Execution** :
Enables you to build[durable agents](/docs/ai/capabilities/durable_execution/overview) that can preserve their progress across transient API failures and application errors or restarts, and handle long-running, asynchronous, and human-in-the-loop workflows with production-grade reliability.
10. 
**Streamed Outputs** :
Provides the ability to[stream](/docs/ai/core-concepts/output#streamed-results) structured output continuously, with immediate validation, ensuring real time access to generated data.
11. 
**Graph Support** :
Provides a powerful way to define[graphs](/docs/ai/graph/graph) using type hints, for use in complex applications where standard control flow can degrade to spaghetti code.

Realistically though, no list is going to be as convincing as [giving it a try](#next-steps) and seeing how it makes you feel!

**Sign up for our newsletter, *The Pydantic Stack*, with updates & tutorials on Pydantic AI, Logfire, and Pydantic:**

Here’s a minimal example of Pydantic AI:

We configure the agent to use [Anthropic's Claude Sonnet 4.6](/docs/ai/api/models/anthropic) model, but you can also set the model when running the agent.

Register static [instructions](/docs/ai/core-concepts/agent#instructions) using a keyword argument to the agent.

[Run the agent](/docs/ai/core-concepts/agent#running-agents) synchronously, starting a conversation with the LLM.

*(This example is complete, it can be run “as is”, assuming you’ve [installed the `pydantic_ai` package](/docs/ai/overview/install))*

The exchange will be very short: Pydantic AI will send the instructions and the user prompt to the LLM, and the model will return a text response.

Not very interesting yet, but we can easily add [tools](/docs/ai/tools-toolsets/tools), [dynamic instructions](/docs/ai/core-concepts/agent#instructions), [structured outputs](/docs/ai/core-concepts/output), or composable [capabilities](/docs/ai/capabilities/overview) to build more powerful agents.

Here’s the same agent with [thinking](/docs/ai/capabilities/thinking) and [web search](/docs/ai/capabilities/web-search) capabilities:

Here is a concise example using Pydantic AI to build a support agent for a bank:

This [agent](/docs/ai/core-concepts/agent) will act as first-tier support in a bank. Agents are generic in the type of dependencies they accept and the type of output they return. In this case, the support agent has type `Agent[SupportDependencies, SupportOutput]`.

Here we configure the agent to use [OpenAI's GPT-5 model](/docs/ai/api/models/openai), you can also set the model when running the agent.

The `SupportDependencies` dataclass is used to pass data, connections, and logic into the model that will be needed when running [instructions](/docs/ai/core-concepts/agent#instructions) and [tool](/docs/ai/tools-toolsets/tools) functions. Pydantic AI's system of dependency injection provides a [type-safe](/docs/ai/core-concepts/agent#static-type-checking) way to customise the behavior of your agents, and can be especially useful when running [unit tests](/docs/ai/guides/testing) and evals.

Static [instructions](/docs/ai/core-concepts/agent#instructions) can be registered with the [`instructions` keyword argument](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.__init__) to the agent.

Dynamic [instructions](/docs/ai/core-concepts/agent#instructions) can be registered with the [`@agent.instructions`](/docs/ai/api/pydantic-ai/agent/#pydantic_ai.agent.Agent.instructions) decorator, and can make use of dependency injection. Dependencies are carried via the [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) argument, which is parameterized with the `deps_type` from above. If the type annotation here is wrong, static type checkers will catch it.

The [`@agent.tool`](/docs/ai/tools-toolsets/tools) decorator let you register functions which the LLM may call while responding to a user. Again, dependencies are carried via [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext), any other arguments become the tool schema passed to the LLM. Pydantic is used to validate these arguments, and errors are passed back to the LLM so it can retry.

The docstring of a tool is also passed to the LLM as the description of the tool. Parameter descriptions are [extracted](/docs/ai/tools-toolsets/tools#function-tools-and-schema) from the docstring and added to the parameter schema sent to the LLM.

[Run the agent](/docs/ai/core-concepts/agent#running-agents) asynchronously, conducting a conversation with the LLM until a final response is reached. Even in this fairly simple case, the agent will exchange multiple messages with the LLM as tools are called to retrieve an output.

The response from the agent will be guaranteed to be a `SupportOutput`. If validation fails [reflection](/docs/ai/core-concepts/agent#reflection-and-self-correction), the agent is prompted to try again.

The output will be validated with Pydantic to guarantee it is a `SupportOutput`, since the agent is generic, it'll also be typed as a `SupportOutput` to aid with static type checking.

In a real use case, you'd add more tools and longer instructions to the agent to extend the context it's equipped with and support it can provide.

This is a simple sketch of a database connection, used to keep the example short and readable. In reality, you'd be connecting to an external database (e.g. PostgreSQL) to get information about customers.

This [Pydantic](https://docs.pydantic.dev) model is used to constrain the structured data returned by the agent. From this simple definition, Pydantic builds the JSON Schema that tells the LLM how to return the data, and performs validation to guarantee the data is correct at the end of the run.

Even a simple agent with just a handful of tools can result in a lot of back-and-forth with the LLM, making it nearly impossible to be confident of what’s going on just from reading the code. To understand the flow of the above runs, we can watch the agent in action using Pydantic Logfire.

To do this, we need to [set up Logfire](/docs/ai/integrations/logfire#using-logfire), and add the following to our code:

Configure the Logfire SDK, this will fail if project is not set up.

This will instrument all Pydantic AI agents used from here on out. To instrument only a specific agent, add an [`Instrumentation`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Instrumentation) entry to the agent's `capabilities=[...]`.

In our demo, `DatabaseConn` uses [`sqlite3`](https://docs.python.org/3/library/sqlite3.html#module-sqlite3) to connect to a PostgreSQL database, so [`logfire.instrument_sqlite3()`](https://logfire.pydantic.dev/docs/integrations/databases/sqlite3/)
is used to log the database queries.

That’s enough to get the following view of your agent in action:

See [Monitoring and Performance](/docs/ai/integrations/logfire) to learn more.

The Pydantic AI documentation is available in the [llms.txt](https://llmstxt.org/) format.
This format is defined in Markdown and suited for LLMs and AI coding assistants and agents.

Two formats are available:

- [`llms.txt`](https://ai.pydantic.dev/llms.txt) : a file containing a brief description
of the project, along with links to the different sections of the documentation. The structure
of this file is described in details[here](https://llmstxt.org/#format) .
- [`llms-full.txt`](https://ai.pydantic.dev/llms-full.txt) : Similar to the`llms.txt` file,
but every link content is included. Note that this file may be too large for some LLMs.

As of today, these files are not automatically leveraged by IDEs or coding agents, but they will use it if you provide a link or the full text.

To try Pydantic AI for yourself, [install it](/docs/ai/overview/install) and follow the instructions [in the examples](/docs/ai/examples/setup).

Read the [docs](/docs/ai/core-concepts/agent) to learn more about building applications with Pydantic AI.

Read the [API Reference](/docs/ai/api/pydantic-ai/agent) to understand Pydantic AI’s interface.

Join [Slack](https://logfire.pydantic.dev/docs/join-slack/) or file an issue on  [GitHub](https://github.com/pydantic/pydantic-ai/issues) if you have any questions.

# Citations

1. Source page: https://pydantic.dev/docs/ai/overview
