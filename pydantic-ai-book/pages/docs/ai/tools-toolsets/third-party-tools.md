---
type: Web Page
title: Third-Party Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/third-party-tools
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Third-Party Tools

Pydantic AI supports integration with various third-party tool libraries, allowing you to leverage existing tool ecosystems in your agents. Third-party tools are also available as capabilities — see Extensibility for the full ecosystem.

See the MCP Client documentation for how to use MCP servers with Pydantic AI as toolsets.

If you’d like to use a tool from LangChain’s community tool library with Pydantic AI, you can use the `tool_from_langchain` convenience method. Note that Pydantic AI will not validate the arguments in this case — it’s up to the model to provide arguments matching the schema specified by the LangChain tool, and up to the LangChain tool to raise an error if the arguments are invalid.

You will need to install the `langchain-community` package and any others required by the tool in question.

Here is how you can use the LangChain `DuckDuckGoSearchRun` tool, which requires the `ddgs` package:

The release date of this game is the 30th of May 2025, which is after the knowledge cutoff for Gemini 2.0 (August 2024).

If you’d like to use multiple LangChain tools or a LangChain toolkit, you can use the `LangChainToolset` toolset which takes a list of LangChain tools:

```
from langchain_community.agent_toolkits import SlackToolkit
from pydantic_ai import Agent
from pydantic_ai.ext.langchain import LangChainToolset
toolkit = SlackToolkit()
toolset = LangChainToolset(toolkit.get_tools())
agent = Agent('openai:gpt-5.2', toolsets=[toolset])
# ...
```
- Function Tools - Basic tool concepts and registration
- Toolsets - Managing collections of tools
- MCP Client - Using MCP servers with Pydantic AI
- LangChain Toolsets - Using LangChain toolsets

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/third-party-tools
