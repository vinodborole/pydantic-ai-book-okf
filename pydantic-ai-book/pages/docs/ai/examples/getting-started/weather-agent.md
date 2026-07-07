---
type: Web Page
title: Weather Agent | Pydantic Docs
resource: https://pydantic.dev/docs/ai/examples/getting-started/weather-agent
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Weather Agent

Example of Pydantic AI with multiple tools which the LLM needs to call in turn to answer a question.

Demonstrates:

- tools
- agent dependencies
- streaming text responses
- Building a Gradio UI for the agent

In this case the idea is a “weather” agent — the user can ask for the weather in multiple locations,
the agent will use the `get_lat_lng` tool to get the latitude and longitude of the locations, then use
the `get_weather` tool to get the weather for those locations.

To run this example properly, you might want to add two extra API keys **(Note if either key is missing, the code will fall back to dummy data, so they’re not required)**:

- A weather API key from tomorrow.io set via `WEATHER_API_KEY`
- A geocoding API key from geocode.maps.co set via `GEO_API_KEY`

With dependencies installed and environment variables set, run:

You can build multi-turn chat applications for your agent with Gradio, a framework for building AI web applications entirely in python. Gradio comes with built-in chat components and agent support so the entire UI will be implemented in a single python file!

Here’s what the UI looks like for the weather agent:

# Citations

1. Source page: https://pydantic.dev/docs/ai/examples/getting-started/weather-agent
