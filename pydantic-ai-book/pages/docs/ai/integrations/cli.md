---
type: Web Page
title: Command Line Interface (CLI) | Pydantic Docs
resource: https://pydantic.dev/docs/ai/integrations/cli
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Command Line Interface (CLI)

**Pydantic AI** comes with a CLI, `clai` (pronounced “clay”). You can use it to chat with various LLMs and quickly get answers, right from the command line, or spin up a uvicorn server to chat with your Pydantic AI agents from your browser.

You can run the `clai` using `uvx`:

Or install `clai` globally with `uv`:

Or with `pip`:

You’ll need to set an environment variable depending on the provider you intend to use.

E.g. if you’re using OpenAI, set the `OPENAI_API_KEY` environment variable:

Then running `clai` will start an interactive session where you can chat with the AI model. Special commands available in interactive mode:

- `/exit`: Exit the session
- `/markdown`: Show the last response in markdown format
- `/multiline`: Toggle multiline input mode (use Ctrl+D to submit)
- `/cp`: Copy the last response to clipboard

| Option | Description | 
|---|---|
| `prompt` | AI prompt for one-shot mode (positional). If omitted, starts interactive mode. | 
| `-m`,`--model` | Model to use in `provider:model`format (e.g.,`openai:gpt-5.2`) | 
| `-a`,`--agent` | Custom agent in `module:variable`format | 
| `-t`,`--code-theme` | Syntax highlighting theme ( `dark`,`light`, or pygments theme) | 
| `--no-stream` | Disable streaming from the model | 
| `-l`,`--list-models` | List all available models and exit | 
| `--version` | Show version and exit | 

You can specify which model to use with the `--model` flag:

(a full list of models available can be printed with `clai --list-models`)

You can specify a custom agent using the `--agent` flag with a module path and variable name:

Then run:

The format must be `module:variable` where:

- `module`is the importable Python module path
- `variable`is the name of the Agent instance in that module

Additionally, you can directly launch CLI mode from an `Agent` instance using `Agent.to_cli_sync()`:

You can also use the async interface with `Agent.to_cli()`:

*(You’ll need to add  asyncio.run(main()) to run main)*

Both `Agent.to_cli()` and `Agent.to_cli_sync()` support a `message_history` parameter, allowing you to continue an existing conversation or provide conversation context:

The CLI will start with the provided conversation history, allowing the agent to refer back to previous exchanges and maintain context throughout the session.

Launch a web-based chat interface by running:

This will start a web server (default: http://127.0.0.1:7932) with a chat interface.

You can also serve an existing agent. For example, if you have an agent defined in `my_agent.py`:

```
from pydantic_ai import Agent
my_agent = Agent('openai:gpt-5.2', instructions='You are a helpful assistant.')
```
Launch the web UI:

| Option | Description | 
|---|---|
| `--agent`,`-a` | Agent to serve in `module:variable`format | 
| `--model`,`-m` | Models to list as options in the UI (repeatable) | 
| `--tool`,`-t` | Native tools to list as options in the UI (repeatable). See available tools. | 
| `--instructions`,`-i` | System instructions. When `--agent`is specified, these are additional to the agent’s existing instructions. | 
| `--host` | Host to bind server (default: 127.0.0.1) | 
| `--port` | Port to bind server (default: 7932) | 
| `--html-source` | URL or file path for the chat UI HTML. | 

When using `--agent`, the agent’s configured model becomes the default. CLI models (`-m`) are additional options. Without `--agent`, the first `-m` model is the default.

The web chat UI can also be launched programmatically using `Agent.to_web()`, see the Web UI documentation.

Run the `web` command with `--help` to see all available options:

# Citations

1. Source page: https://pydantic.dev/docs/ai/integrations/cli
