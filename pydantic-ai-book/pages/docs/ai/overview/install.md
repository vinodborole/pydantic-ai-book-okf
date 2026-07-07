---
type: Web Page
title: Installation | Pydantic Docs
resource: https://pydantic.dev/docs/ai/overview/install
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Installation

Pydantic AI is available on PyPI as `pydantic-ai` so installation is as simple as:

(Requires Python 3.10+)

This installs the `pydantic_ai` package, core dependencies, and libraries required to use the OpenAI, Anthropic, and Google models, plus the CLI, MCP, Evals, Web UI, Retries, and Logfire integrations.
To use any other models or integrations, add the relevant extras to your install command, e.g. `pydantic-ai[bedrock,temporal]`. Alternatively, you can install the `pydantic-ai-slim` package with only the extras you need.

Pydantic AI has an excellent (but completely optional) integration with Pydantic Logfire to help you view and understand agent runs.

Logfire comes included with `pydantic-ai` (but not the “slim” version), so you can typically start using it immediately by following the Logfire setup docs.

We distribute the `pydantic_ai_examples` directory as a separate PyPI package (`pydantic-ai-examples`) to make examples extremely easy to customize and run.

To install examples, use the `examples` optional group:

To run the examples, follow instructions in the examples docs.

If you know which model you’re going to use and want to avoid installing superfluous packages, you can use the `pydantic-ai-slim` package.
For example, if you’re using just `OpenAIChatModel`, you would run:

`pydantic-ai-slim` has the following optional groups:

- `logfire`— installs Pydantic Logfire dependency- `logfire`PyPI ↗
- `evals`— installs Pydantic Evals dependency- `pydantic-evals`PyPI ↗
- `openai`— installs OpenAI Model dependency- `openai`PyPI ↗
- `google`— installs Google Model dependency- `google-genai`PyPI ↗
- `anthropic`— installs Anthropic Model dependency- `anthropic`PyPI ↗
- `groq`— installs Groq Model dependency- `groq`PyPI ↗
- `mistral`— installs Mistral Model dependency- `mistralai`PyPI ↗
- `cohere`- installs Cohere Model dependency- `cohere`PyPI ↗
- `bedrock`- installs Bedrock Model dependency- `boto3`PyPI ↗
- `xai`- installs xAI Model dependency- `xai-sdk`PyPI ↗
- `openrouter`- installs the OpenRouter dependency- `openai`PyPI ↗
- `huggingface`- installs Hugging Face Model dependency- `huggingface-hub`PyPI ↗
- `sentence-transformers`- installs Sentence Transformers Embedding Model dependency- `sentence-transformers`PyPI ↗
- `voyageai`- installs VoyageAI Embedding Model dependency- `voyageai`PyPI ↗
- `duckduckgo`- installs DuckDuckGo Search Tool dependency- `ddgs`PyPI ↗
- `tavily`- installs Tavily Search Tool dependency- `tavily-python`PyPI ↗
- `exa`- installs Exa Search Tool dependency- `exa-py`PyPI ↗
- `web-fetch`- installs Web Fetch Tool dependency- `markdownify`PyPI ↗
- `cli`- installs CLI dependencies- `rich`PyPI ↗,- `prompt-toolkit`PyPI ↗, and- `argcomplete`PyPI ↗
- `mcp`- installs MCP dependency- `fastmcp-slim[client]`PyPI ↗
- `ui`- installs UI Event Streams dependency- `starlette`PyPI ↗
- `web`- installs Web UI dependencies- `starlette`PyPI ↗,- `httpx`PyPI ↗, and- `uvicorn`PyPI ↗
- `ag-ui`- installs AG-UI Event Stream Protocol dependencies- `ag-ui-protocol`PyPI ↗ and- `starlette`PyPI ↗
- `retries`- installs HTTP Retries dependency- `tenacity`PyPI ↗
- `temporal`- installs Temporal Durable Execution dependency- `temporalio`PyPI ↗
- `dbos`- installs DBOS Durable Execution dependency- `dbos`PyPI ↗
- `prefect`- installs Prefect Durable Execution dependency- `prefect`PyPI ↗
- `spec`- installs AgentSpec dependencies- `pyyaml`PyPI ↗ and- `pydantic-handlebars`PyPI ↗

You can also install dependencies for multiple models and use cases, for example:

# Citations

1. Source page: https://pydantic.dev/docs/ai/overview/install
