---
type: Web Page
title: Chat App with FastAPI | Pydantic Docs
resource: https://pydantic.dev/docs/ai/examples/conversational-agents/chat-app
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Chat App with FastAPI

Simple chat app example build with FastAPI.

Demonstrates:

This demonstrates storing chat history between requests and using it to give the model context for new responses.

Most of the complex logic here is between `chat_app.py` which streams the response to the browser,
and `chat_app.ts` which renders messages in the browser.

With [dependencies installed and environment variables set](/docs/ai/examples/setup#usage), run:

Then open the app at [localhost:8000](http://localhost:8000).

Python code that runs the chat app:

Simple HTML page to render the app:

TypeScript to handle rendering the messages, to keep this simple (and at the risk of offending frontend developers) the typescript code is passed to the browser as plain text and transpiled in the browser.

# Citations

1. Source page: https://pydantic.dev/docs/ai/examples/conversational-agents/chat-app
