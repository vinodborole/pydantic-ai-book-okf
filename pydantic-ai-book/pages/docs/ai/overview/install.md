---
type: Web Page
title: Installation | Pydantic Docs
resource: https://pydantic.dev/docs/ai/overview/install
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Installation

Pydantic AI is available on PyPI as [`pydantic-ai`](https://pypi.org/project/pydantic-ai/) so installation is as simple as:

(Requires Python 3.10+)

This installs the `pydantic_ai` package, core dependencies, and libraries required to use the OpenAI, Anthropic, and Google models, plus the [CLI](/docs/ai/integrations/cli), [MCP](/docs/ai/mcp/client), [Evals](/docs/ai/evals/evals), [Web UI](/docs/ai/integrations/ui/overview), [Retries](/docs/ai/models/http-request-retries), and [Logfire](/docs/ai/integrations/logfire) integrations.
To use any other models or integrations, add the relevant extras to your install command, e.g. `pydantic-ai[bedrock,temporal]`. Alternatively, you can install the [`pydantic-ai-slim`](#slim-install) package with only the extras you need.

Pydantic AI has an excellent (but completely optional) integration with [Pydantic Logfire](https://pydantic.dev/logfire) to help you view and understand agent runs.

Logfire comes included with `pydantic-ai` (but not the [“slim” version](#slim-install)), so you can typically start using it immediately by following the [Logfire setup docs](/docs/ai/integrations/logfire#using-logfire).

We distribute the [`pydantic_ai_examples`](https://github.com/pydantic/pydantic-ai/tree/main/examples/pydantic_ai_examples) directory as a separate PyPI package ([`pydantic-ai-examples`](https://pypi.org/project/pydantic-ai-examples/)) to make examples extremely easy to customize and run.

To install examples, use the `examples` optional group:

To run the examples, follow instructions in the [examples docs](/docs/ai/examples/setup).

If you know which model you’re going to use and want to avoid installing superfluous packages, you can use the [`pydantic-ai-slim`](https://pypi.org/project/pydantic-ai-slim/) package.
For example, if you’re using just [`OpenAIChatModel`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIChatModel), you would run:

`pydantic-ai-slim` has the following optional groups:

- `logfire` — installs[Pydantic Logfire](/docs/ai/integrations/logfire) dependency`logfire`[PyPI ↗](https://pypi.org/project/logfire)
- `evals` — installs[Pydantic Evals](/docs/ai/evals/evals) dependency`pydantic-evals`[PyPI ↗](https://pypi.org/project/pydantic-evals)
- `openai` — installs[OpenAI Model](/docs/ai/models/openai) dependency`openai`[PyPI ↗](https://pypi.org/project/openai)
- `google` — installs[Google Model](/docs/ai/models/google) dependency`google-genai`[PyPI ↗](https://pypi.org/project/google-genai)
- `anthropic` — installs[Anthropic Model](/docs/ai/models/anthropic) dependency`anthropic`[PyPI ↗](https://pypi.org/project/anthropic)
- `groq` — installs[Groq Model](/docs/ai/models/groq) dependency`groq`[PyPI ↗](https://pypi.org/project/groq)
- `mistral` — installs[Mistral Model](/docs/ai/models/mistral) dependency`mistralai`[PyPI ↗](https://pypi.org/project/mistralai)
- `cohere` - installs[Cohere Model](/docs/ai/models/cohere) dependency`cohere`[PyPI ↗](https://pypi.org/project/cohere)
- `bedrock` - installs[Bedrock Model](/docs/ai/models/bedrock) dependency`boto3`[PyPI ↗](https://pypi.org/project/boto3)
- `bedrock-mantle` - installs[Bedrock Mantle Model](/docs/ai/models/bedrock#bedrock-mantle) dependencies`openai`[PyPI ↗](https://pypi.org/project/openai) and`botocore`[PyPI ↗](https://pypi.org/project/botocore)
- `xai` - installs[xAI Model](/docs/ai/models/xai) dependency`xai-sdk`[PyPI ↗](https://pypi.org/project/xai-sdk)
- `openrouter` - installs the[OpenRouter](/docs/ai/models/openrouter) dependency`openai`[PyPI ↗](https://pypi.org/project/openai)
- `zai` - installs the[Z.AI](/docs/ai/models/zai) dependency`openai`[PyPI ↗](https://pypi.org/project/openai)
- `huggingface` - installs[Hugging Face Model](/docs/ai/models/huggingface) dependency`huggingface-hub`[PyPI ↗](https://pypi.org/project/huggingface-hub)
- `sentence-transformers` - installs[Sentence Transformers Embedding Model](/docs/ai/guides/embeddings#sentence-transformers-local) dependency`sentence-transformers`[PyPI ↗](https://pypi.org/project/sentence-transformers)
- `voyageai` - installs[VoyageAI Embedding Model](/docs/ai/guides/embeddings#voyageai) dependency`voyageai`[PyPI ↗](https://pypi.org/project/voyageai)
- `duckduckgo` - installs[DuckDuckGo Search Tool](/docs/ai/tools-toolsets/common-tools#duckduckgo-search-tool) dependency`ddgs`[PyPI ↗](https://pypi.org/project/ddgs)
- `tavily` - installs[Tavily Search Tool](/docs/ai/tools-toolsets/common-tools#tavily-search-tool) dependency`tavily-python`[PyPI ↗](https://pypi.org/project/tavily-python)
- `exa` - installs[Exa Search Tool](/docs/ai/tools-toolsets/common-tools#exa-search-tool) dependency`exa-py`[PyPI ↗](https://pypi.org/project/exa-py)
- `web-fetch` - installs[Web Fetch Tool](/docs/ai/tools-toolsets/common-tools#web-fetch-tool) dependency`markdownify`[PyPI ↗](https://pypi.org/project/markdownify)
- `cli` - installs[CLI](/docs/ai/integrations/cli) dependencies`rich`[PyPI ↗](https://pypi.org/project/rich) ,`prompt-toolkit`[PyPI ↗](https://pypi.org/project/prompt-toolkit) , and`argcomplete`[PyPI ↗](https://pypi.org/project/argcomplete)
- `mcp` - installs[MCP](/docs/ai/mcp/client) dependency`fastmcp-slim[client]`[PyPI ↗](https://pypi.org/project/fastmcp-slim)
- `ui` - installs[UI Event Streams](/docs/ai/integrations/ui/overview) dependency`starlette`[PyPI ↗](https://pypi.org/project/starlette)
- `web` - installs[Web UI](/docs/ai/integrations/ui/overview) dependencies`starlette`[PyPI ↗](https://pypi.org/project/starlette) ,`httpx`[PyPI ↗](https://pypi.org/project/httpx) , and`uvicorn`[PyPI ↗](https://pypi.org/project/uvicorn)
- `ag-ui` - installs[AG-UI Event Stream Protocol](/docs/ai/integrations/ui/ag-ui) dependencies`ag-ui-protocol`[PyPI ↗](https://pypi.org/project/ag-ui-protocol) and`starlette`[PyPI ↗](https://pypi.org/project/starlette)
- `retries` - installs[HTTP Retries](/docs/ai/models/http-request-retries) dependency`tenacity`[PyPI ↗](https://pypi.org/project/tenacity)
- `temporal` - installs[Temporal Durable Execution](/docs/ai/capabilities/durable_execution/temporal) dependency`temporalio`[PyPI ↗](https://pypi.org/project/temporalio)
- `dbos` - installs[DBOS Durable Execution](/docs/ai/capabilities/durable_execution/dbos) dependency`dbos`[PyPI ↗](https://pypi.org/project/dbos)
- `prefect` - installs[Prefect Durable Execution](/docs/ai/capabilities/durable_execution/prefect) dependency`prefect`[PyPI ↗](https://pypi.org/project/prefect)
- `spec` - installs[AgentSpec](/docs/ai/core-concepts/agent-spec) dependencies`pyyaml`[PyPI ↗](https://pypi.org/project/PyYAML) and`pydantic-handlebars`[PyPI ↗](https://pypi.org/project/pydantic-handlebars)

You can also install dependencies for multiple models and use cases, for example:

# Citations

1. Source page: https://pydantic.dev/docs/ai/overview/install
