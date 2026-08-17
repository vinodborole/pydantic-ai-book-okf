---
type: Web Page
title: Pydantic AI | Pydantic Docs
description: 'How Python does AI: agents, realtime voice, image generation, embeddings.
  Every model, every interface, typed end to end.'
resource: https://pydantic.dev/docs/ai/overview
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Pydantic AI

*How Python does AI*

Agents, realtime voice, image generation, embeddings. Every model, every interface, typed end to end.

**Pydantic AI** is the Python AI SDK: a typed, [extensible](/docs/ai/guides/extensibility/) agent loop with [every model](/docs/ai/models/overview/) a string swap away. The same agent [runs everywhere you need it](/docs/ai/overview/interfaces/): behind a [web frontend](/docs/ai/integrations/ui/overview/), in the [terminal](/docs/ai/integrations/cli/), on a [voice call](/docs/ai/realtime/overview/), on a [durable background queue](/docs/ai/capabilities/durable_execution/overview/), or as a plain object you call [`run()`](/docs/ai/core-concepts/agent/#running-agents) on. [Image generation](/docs/ai/capabilities/image-generation/) and [embeddings](/docs/ai/guides/embeddings/) come in the same box.

**[Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/)** has everything an agent needs for complex, long-running work, snapped on as [capabilities](/docs/ai/capabilities/overview/), from [memory](https://pydantic.dev/docs/ai/harness/memory/), [sub-agents](https://pydantic.dev/docs/ai/harness/subagents/), and [context management](https://pydantic.dev/docs/ai/harness/compaction/) to a complete [coding agent](https://pydantic.dev/docs/ai/harness/coder/).

From simple typed data extraction to complex, long-running multi-agent collaboration, Pydantic AI and [Pydantic AI Harness](https://pydantic.dev/docs/ai/harness/) have got you covered.

A complete coding agent in your terminal: workspace-rooted [file access](https://pydantic.dev/docs/ai/harness/filesystem/), allowlisted [shell](https://pydantic.dev/docs/ai/harness/shell/), [repo orientation](https://pydantic.dev/docs/ai/harness/repo-context/), [planning](https://pydantic.dev/docs/ai/harness/planning/), and [context management](https://pydantic.dev/docs/ai/harness/compaction/) that survives long sessions. Here with [web search](/docs/ai/capabilities/web-search/) and a second-opinion [advisor](https://pydantic.dev/docs/ai/harness/advisor/) snapped on alongside:

```
from pydantic_ai import Agent
from pydantic_ai.capabilities import WebSearch
from pydantic_ai_harness import Advisor, Coder
agent = Agent(
    'anthropic:claude-fable-5',
    capabilities=[
        Coder(),  # files, shell, repo context, planning, sub-agents, context management
        WebSearch(),  # look up docs and error messages on the web
        Advisor('openai:gpt-5.6-sol'),  # a second opinion from another model when stuck
    ],
)
agent.to_cli_sync()
```
[`Coder`](https://pydantic.dev/docs/ai/harness/coder/) is a regular [combined capability](/docs/ai/capabilities/custom/#composition-and-middleware-semantics), not a black box: use it whole, or use the blocks it bundles directly; the two are equivalent:

```
capabilities = [
    FileSystem('.'), Shell(cwd='.'), RepoContext(), Planning(), SubAgents(...),
    ClearToolResults(), WarnNearLimits(), ToolOutputLimits(),
]
```
Run the file and you’re chatting with the agent in your terminal. To try it before writing any code, run the exported [`coder_agent`](https://pydantic.dev/docs/ai/harness/coder/#api-reference) with [`clai`](/docs/ai/integrations/cli/#custom-agents) (the Pydantic AI CLI), via [`uvx`](https://docs.astral.sh/uv/guides/tools/):

Give the agent an [output type](/docs/ai/core-concepts/output/) and [tools](/docs/ai/tools-toolsets/tools/), and every run comes back validated and typed:

The [`@agent.tool`](/docs/ai/tools-toolsets/tools/) function receives a [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) that carries your [dependencies](/docs/ai/core-concepts/dependencies/) in; the rest of its signature and its docstring become the tool schema, arguments are [validated](/docs/ai/tools-toolsets/tools/#function-tools-and-schema) before your code runs, and the run is guaranteed to return a `Sentiment`, so your IDE, type checker, and the LLM all agree on the shape.

**Build this →** [Agents](/docs/ai/core-concepts/agent/), [Function Tools](/docs/ai/tools-toolsets/tools/), and [Structured Output](/docs/ai/core-concepts/output/)

Put the same agent on a live voice session, [tools](/docs/ai/realtime/tools/) and [capabilities](/docs/ai/realtime/capabilities/) included:

```
import asyncio
from pydantic_ai import Agent
from pydantic_ai.capabilities import MCP
agent = Agent(
    instructions='You are a helpful voice assistant.',
    capabilities=[MCP('https://internal.example.com/mcp')],  # capabilities work in voice too
)
@agent.tool_plain
def order_status(order_id: str) -> str:
    """Look up the status of an order."""
    return f'Order {order_id}: shipped, arriving Thursday.'
async with agent.realtime('openai:gpt-realtime-2.1').session() as session:
    microphone = asyncio.create_task(stream_microphone(session))  # chunks → session.send_audio()
    speaker = asyncio.create_task(play_audio(session.stream_audio()))  # model audio → your speaker
    async for part in session.stream_transcripts():
        print(f'{part.speaker}: {part.transcript}')
```
The model calls your tools mid-conversation while it keeps talking, and every session is [instrumented](/docs/ai/integrations/logfire/); voice is just another frontend, on OpenAI Realtime, Gemini Live, Azure, and xAI Grok Voice.

**Build this →** [Realtime Voice](/docs/ai/realtime/overview/), starting from the [voice assistant example](/docs/ai/examples/realtime/realtime-voice/)

Attach [`TemporalDurability`](/docs/ai/capabilities/durable_execution/temporal/) and the same agent runs inside a [Temporal](/docs/ai/capabilities/durable_execution/temporal/) workflow: every model and tool call becomes a durable activity, so a run working through a background queue survives restarts, failures, and long waits:

[DBOS](/docs/ai/capabilities/durable_execution/dbos/) and [Prefect](/docs/ai/capabilities/durable_execution/prefect/) attach the same way, first-party and co-maintained, with [Restate, Kitaru, and Airflow](/docs/ai/capabilities/durable_execution/overview/) integrations besides.

**Build this →** [Durable Execution](/docs/ai/capabilities/durable_execution/overview/)

Ask for an image and make it the run’s typed [output](/docs/ai/core-concepts/output/):

[Provider-native generation](/docs/ai/tools-toolsets/native-tools/#image-generation-tool) on models that support it (like this one), a [subagent fallback](/docs/ai/capabilities/image-generation/) you can configure for the rest, and a [standalone image API](https://github.com/pydantic/pydantic-ai/pull/5357) on the way.

**Build this →** [Image Generation](/docs/ai/capabilities/image-generation/)

Embed documents and queries for semantic search or a [RAG pipeline](/docs/ai/examples/data-analytics/rag/):

Seven providers behind one typed API, [instrumented](/docs/ai/integrations/logfire/) like everything else. It lives next to the agent that will use the results.

**Build this →** [Embeddings](/docs/ai/guides/embeddings/), then the [RAG example](/docs/ai/examples/data-analytics/rag/)

- 
**Any model, one Python API.**[Virtually every model and provider](/docs/ai/models/overview/) (OpenAI, Anthropic, Google, Bedrock, Azure AI Foundry, Groq, Mistral, xAI, Ollama, and dozens more), swappable with a string, or through the[Pydantic AI Gateway](/docs/ai/overview/gateway/) : one key for all of them, with failover and cost monitoring built in. No flagship feature is locked to one vendor.
- 
**Typed end to end.**[Structured outputs](/docs/ai/core-concepts/output/) , typed[dependency injection](/docs/ai/core-concepts/dependencies/) ,[typed tools](/docs/ai/tools-toolsets/tools/) : your IDE, type checker, and coding agent all know what your agent returns, moving whole classes of errors from runtime to write-time. When plain control flow isn’t enough,[Pydantic Graph](/docs/ai/graph/graph/) brings the same typing to graph-based workflows.
- 
**Measured, not vibes.** OpenTelemetry-native[instrumentation](/docs/ai/integrations/logfire/) works with any OTel backend; one line lights up[Pydantic Logfire](https://pydantic.dev/logfire) for real-time debugging, tracing, and cost tracking backed by[genai-prices](https://github.com/pydantic/genai-prices) .[Pydantic Evals](/docs/ai/evals/evals/) tests agent behavior the way pytest tests code.
- 
**Batteries, composably.** One primitive, the[capability](/docs/ai/capabilities/overview/) , bundles[tools](/docs/ai/tools-toolsets/tools/) ,[instructions](/docs/ai/core-concepts/agent/#instructions) ,[hooks](/docs/ai/core-concepts/hooks/) , and[model settings](/docs/ai/core-concepts/agent/#model-run-settings) into reusable units. Core ships fundamentals like[MCP](/docs/ai/capabilities/mcp/) and[web search](/docs/ai/capabilities/web-search/) , the[Harness](https://pydantic.dev/docs/ai/harness/) ships everything else, and complete agents like[Coder](https://pydantic.dev/docs/ai/harness/coder/) and[Researcher](https://pydantic.dev/docs/ai/harness/researcher/) are just capabilities composed: they come apart the way they went together. Or skip code entirely with[YAML/JSON agent specs](/docs/ai/core-concepts/agent-spec/) .
- 
**[Every interface](/docs/ai/overview/interfaces/).** One agent definition runs as a[CLI](/docs/ai/integrations/cli/) , a[built-in web chat](/docs/ai/guides/web/) , or[realtime speech](/docs/ai/realtime/overview/) ;[UI event streams](/docs/ai/integrations/ui/overview/) (AG-UI, Vercel AI) connect it to your own frontend or anything else; and[ACP](https://pydantic.dev/docs/ai/harness/acp/)*(experimental)* serves it as an editor agent.
- 
**Durable execution.** First-party, co-maintained[durable execution](/docs/ai/capabilities/durable_execution/overview/) on Temporal, DBOS, or Prefect, with[Restate, Kitaru, and Airflow](/docs/ai/capabilities/durable_execution/overview/) integrations and more coming. Agents survive restarts and run for days on the engine you already operate, with[human-in-the-loop approval](/docs/ai/tools-toolsets/deferred-tools/#human-in-the-loop-tool-approval) built in.

Built by the [Pydantic](https://docs.pydantic.dev) team: [Pydantic Validation](https://pydantic.dev/docs/) is the validation layer of the OpenAI SDK, the Anthropic SDK, the Google ADK, LangChain, and most of the AI ecosystem (and the foundation FastAPI was built on). Pydantic AI brings that same feeling to agents.

**Sign up for our newsletter, *The Pydantic Stack*, with updates & tutorials on Pydantic AI, Logfire, and Pydantic:**

Here’s a support agent for a bank, showing several features working together: [dependency injection](/docs/ai/core-concepts/dependencies/) carrying a database connection into instructions and tools, [function tools](/docs/ai/tools-toolsets/tools/) the model calls, [structured output](/docs/ai/core-concepts/output/) validated on every run, a reusable [capability](/docs/ai/capabilities/overview/) bundling the customer context, and an [on-demand capability](/docs/ai/capabilities/on-demand/) the model loads only when the conversation calls for it:

The `SupportDependencies` dataclass is used to pass data, connections, and logic into the model that will be needed when running [instructions](/docs/ai/core-concepts/agent/#instructions) and [tool](/docs/ai/tools-toolsets/tools/) functions. Pydantic AI's system of [dependency injection](/docs/ai/core-concepts/dependencies/) provides a [type-safe](/docs/ai/core-concepts/agent/#static-type-checking) way to customise the behavior of your agents, and can be especially useful when running [unit tests](/docs/ai/guides/testing/) and evals.

This is a simple sketch of a database connection, used to keep the example short and readable. In reality, you'd be connecting to an external database (e.g. PostgreSQL) to get information about customers.

This [Pydantic](https://docs.pydantic.dev) model is used to constrain the structured data returned by the agent. From this simple definition, Pydantic builds the JSON Schema that tells the LLM how to return the data, and performs validation to guarantee the data is correct at the end of the run.

A [`Capability`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Capability) bundles related instructions and tools into one reusable unit: the same primitive behind [built-in capabilities](/docs/ai/capabilities/overview/) like [web search](/docs/ai/capabilities/web-search/) and everything in the [Harness](https://pydantic.dev/docs/ai/harness/). This one carries the customer context; you could drop it into any other agent's `capabilities` list as-is.

Dynamic [instructions](/docs/ai/core-concepts/agent/#instructions) can make use of dependency injection. Dependencies are carried via the [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) argument, which is parameterized with the `deps_type` from above. If the type annotation here is wrong, static type checkers will catch it.

The [`tool`](/docs/ai/tools-toolsets/tools/) decorator registers a function whose signature becomes a tool the LLM may call while responding to a user. Again, dependencies are carried via [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext); any other arguments become the tool schema passed to the LLM. Pydantic is used to validate these arguments, and errors are passed back to the LLM so it can retry.

The docstring of a tool is also passed to the LLM as the description of the tool. Parameter descriptions are [extracted](/docs/ai/tools-toolsets/tools/#function-tools-and-schema) from the docstring and added to the parameter schema sent to the LLM.

`defer_loading=True` makes this an [on-demand capability](/docs/ai/capabilities/on-demand/), the same shape as an [Agent Skill](/docs/ai/capabilities/on-demand/#loading-skills-from-markdown-files). It collapses to a one-line catalog entry in the prompt, and its tools stay hidden until the model decides it's relevant and loads it with the framework-managed `load_capability` tool.

This [agent](/docs/ai/core-concepts/agent/) will act as first-tier support in a bank. Agents are generic in the type of dependencies they accept and the type of output they return. In this case, the support agent has type `Agent[SupportDependencies, SupportOutput]`.

Here we configure the agent to use [OpenAI's GPT-5.6 Sol](/docs/ai/api/models/openai/) model; you can also set the model when running the agent.

The response from the agent will be guaranteed to be a `SupportOutput`. Since the agent is generic, it'll also be typed as a `SupportOutput` to aid with static type checking. If validation fails, the agent is [prompted to try again](/docs/ai/core-concepts/agent/#reflection-and-self-correction).

Mount the capabilities on the agent. More [capabilities](/docs/ai/capabilities/overview/), like [web search](/docs/ai/capabilities/web-search/) or anything from the [Harness](https://pydantic.dev/docs/ai/harness/), snap on alongside them in the same list.

In a real use case, you'd add more tools and longer instructions to the agent to extend the context it's equipped with and support it can provide.

[Run the agent](/docs/ai/core-concepts/agent/#running-agents) asynchronously, conducting a conversation with the LLM until a final response is reached. Even in this fairly simple case, the agent will exchange multiple messages with the LLM as tools are called to retrieve an output.

This turn exercises the deferred capability: the model sees the `refunds` catalog entry, calls `load_capability` with `id='refunds'`, and only then gets the `refund_status` tool to answer with: [on-demand loading](/docs/ai/capabilities/on-demand/) in action.

The [dependencies](/docs/ai/core-concepts/dependencies/) dataclass carries the database connection into [instructions](/docs/ai/core-concepts/agent/#instructions) and [tools](/docs/ai/tools-toolsets/tools/) with full type safety: swap in a test double and the same agent runs in [unit tests](/docs/ai/guides/testing/) and evals. And because the customer context is a [capability](/docs/ai/capabilities/overview/), it composes: the same unit drops into a voice agent or a web app unchanged.

Pydantic AI is [OpenTelemetry](https://opentelemetry.io/)-native: the [Instrumentation](/docs/ai/capabilities/instrumentation/) capability emits standard OTel spans for every model call and tool call, and [any OTLP backend works](/docs/ai/integrations/logfire/#alternative-observability-backends). The easiest setup is the [`logfire` SDK](/docs/ai/integrations/logfire/#using-logfire), which speaks plain OpenTelemetry and can point at [Pydantic Logfire](https://pydantic.dev/logfire) or any other backend.

Even a simple agent with just a handful of tools can result in a lot of back-and-forth with the LLM, making it nearly impossible to be confident of what’s going on just from reading the code. To watch the runs above in action, [set up Logfire](/docs/ai/integrations/logfire/#using-logfire) and add the following to the code:

Configure the Logfire SDK, this will fail if project is not set up.

This will instrument all Pydantic AI agents used from here on out. To instrument only a specific agent, add an [`Instrumentation`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Instrumentation) entry to the agent's `capabilities=[...]`.

In our demo, `DatabaseConn` uses [`sqlite3`](https://docs.python.org/3/library/sqlite3.html#module-sqlite3) to connect to a PostgreSQL database, so [`logfire.instrument_sqlite3()`](https://logfire.pydantic.dev/docs/integrations/databases/sqlite3/)
is used to log the database queries.

That’s enough to get the following view of your agent in action:

See [Monitoring and Performance](/docs/ai/integrations/logfire/) to learn more.

The Pydantic AI documentation is available in the [llms.txt](https://llmstxt.org/) format.
This format is defined in Markdown and suited for LLMs and AI coding assistants and agents.

Two formats are available:

- [`llms.txt`](https://ai.pydantic.dev/llms.txt) : a file containing a brief description
of the project, along with links to the different sections of the documentation. The structure
of this file is described in details[here](https://llmstxt.org/#format) .
- [`llms-full.txt`](https://ai.pydantic.dev/llms-full.txt) : Similar to the`llms.txt` file,
but every link content is included. Note that this file may be too large for some LLMs.

As of today, these files are not automatically leveraged by IDEs or coding agents, but they will use it if you provide a link or the full text.

**Run something right now.** One command puts a complete [coding agent](https://pydantic.dev/docs/ai/harness/coder/) in your terminal:

Or [install Pydantic AI](/docs/ai/overview/install/), pick a [model](/docs/ai/models/overview/), and put your own coding agent to work: install the [Pydantic AI skill](/docs/ai/overview/coding-agent-skills/) to give it up-to-date framework knowledge, point it at the [examples](/docs/ai/examples/setup/) and the [Harness index](https://pydantic.dev/docs/ai/harness/), and tell it what you’d like to build.

**See what your agent did.** [Instrument it](/docs/ai/integrations/logfire/): one line of setup, and every model call and tool call shows up. It’s standard OpenTelemetry: [Pydantic Logfire](https://pydantic.dev/logfire) is the easiest way to look, any OTLP backend works.

**Go deeper.** The [Agents guide](/docs/ai/core-concepts/agent/) is the core walkthrough; the [API Reference](/docs/ai/api/pydantic-ai/agent/) covers the full interface; the [Harness](https://pydantic.dev/docs/ai/harness/) has the batteries.

# Citations

1. Source page: https://pydantic.dev/docs/ai/overview
