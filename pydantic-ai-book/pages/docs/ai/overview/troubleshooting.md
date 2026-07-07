---
type: Web Page
title: Troubleshooting | Pydantic Docs
resource: https://pydantic.dev/docs/ai/overview/troubleshooting
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Troubleshooting

Below are suggestions on how to fix some common errors you might encounter while using Pydantic AI. If the issue you’re experiencing is not listed below or addressed in the documentation, please feel free to ask in the Pydantic Slack or create an issue on GitHub.

**Modern Jupyter/IPython (7.0+)**: This environment supports top-level `await` natively. You can use `Agent.run()` directly in notebook cells without additional setup:

```
from pydantic_ai import Agent
agent = Agent('openai:gpt-5.2')
result = await agent.run('Who let the dogs out?')
```
**Legacy environments or specific integrations**: If you encounter event loop conflicts, use `nest-asyncio`:

```
import nest_asyncio
from pydantic_ai import Agent
nest_asyncio.apply()
agent = Agent('openai:gpt-5.2')
result = agent.run_sync('Who let the dogs out?')
```
**Note**: This also applies to Google Colab and Marimo environments.

If you’re running into issues with setting the API key for your model, visit the Models page to learn more about how to set an environment variable and/or pass in an `api_key` argument.

You can use custom `httpx` clients in your models in order to access specific requests, responses, and headers at runtime.

It’s particularly helpful to use `logfire`’s HTTPX integration to monitor the above.

# Citations

1. Source page: https://pydantic.dev/docs/ai/overview/troubleshooting
