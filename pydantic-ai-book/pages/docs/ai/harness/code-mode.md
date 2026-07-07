---
type: Web Page
title: Code Mode | Pydantic Docs
resource: https://pydantic.dev/docs/ai/harness/code-mode
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Code Mode

Code mode is one of the capabilities in **Pydantic AI Harness**, the official capability library for Pydantic AI. The full docs live in the harness repo — this page is a short intro.

`CodeMode` wraps your tools into a single `run_code` tool powered by our Monty sandbox. The model writes Python that calls multiple tools with loops, conditionals, variables, and `asyncio.gather` — all inside one tool call.

Standard tool calling requires one model round-trip per tool call. An agent that needs to fetch 10 items and process each one makes 11+ model calls — slow, expensive, and context-heavy. Code mode collapses that into one.

```
from pydantic_ai import Agent
from pydantic_ai_harness import CodeMode
agent = Agent('anthropic:claude-sonnet-4-6', capabilities=[CodeMode()])
@agent.tool_plain
def get_weather(city: str) -> dict:
    """Get current weather for a city."""
    return {'city': city, 'temp_f': 72, 'condition': 'sunny'}
@agent.tool_plain
def convert_temp(fahrenheit: float) -> float:
    """Convert Fahrenheit to Celsius."""
    return round((fahrenheit - 32) * 5 / 9, 1)
result = agent.run_sync("What's the weather in Paris and Tokyo, in Celsius?")
print(result.output)
```
The model writes code like:

```
paris, tokyo = await asyncio.gather(
    get_weather(city='Paris'),
    get_weather(city='Tokyo'),
)
paris_c = await convert_temp(fahrenheit=paris['temp_f'])
tokyo_c = await convert_temp(fahrenheit=tokyo['temp_f'])
{'paris': paris_c, 'tokyo': tokyo_c}
```
See the Code Mode README in the harness repo for selective tool sandboxing, metadata-based selection, return value handling, REPL state, observability, sandbox restrictions, the full API, and agent spec usage.

# Citations

1. Source page: https://pydantic.dev/docs/ai/harness/code-mode
