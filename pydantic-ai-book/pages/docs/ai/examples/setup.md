---
type: Web Page
title: Setup | Pydantic Docs
resource: https://pydantic.dev/docs/ai/examples/setup
timestamp: '2026-07-09T12:16:42.049694+00:00'
---

# Setup

Here we include some examples of how to use Pydantic AI and what it can do.

These examples are distributed with `pydantic-ai` so you can run them either by cloning the [pydantic-ai repo](https://github.com/pydantic/pydantic-ai) or by simply installing `pydantic-ai` from PyPI with `pip` or `uv`.

Either way you’ll need to install extra dependencies to run some examples, you just need to install the `examples` optional dependency group.

If you’ve installed `pydantic-ai` via pip/uv, you can install the extra dependencies with:

If you clone the repo, you should instead use `uv sync --extra examples` to install extra dependencies.

These examples will need you to set up authentication with one or more of the LLMs, see the [model configuration](/docs/ai/models/overview) docs for details on how to do this.

TL;DR: in most cases you’ll need to set one of the following environment variables:

To run the examples (this will work whether you installed `pydantic_ai`, or cloned the repo), run:

For example, to run the very simple [ pydantic_model](/docs/ai/examples/pydantic-model) example:

If you like one-liners and you’re using uv, you can run a pydantic-ai example with zero setup:

You’ll probably want to edit examples in addition to just running them. You can copy the examples to a new directory with:

# Citations

1. Source page: https://pydantic.dev/docs/ai/examples/setup
