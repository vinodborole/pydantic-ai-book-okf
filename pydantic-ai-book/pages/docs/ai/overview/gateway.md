---
type: Web Page
title: Pydantic AI Gateway | Pydantic Docs
resource: https://pydantic.dev/docs/ai/overview/gateway
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Pydantic AI Gateway

**Pydantic AI Gateway** is a unified interface for accessing multiple AI providers with a single key, managed through Pydantic Logfire. Features include built-in OpenTelemetry observability, real-time cost monitoring, failover management, and native integration with the other tools in the Pydantic stack.

Sign up at logfire.pydantic.dev.

To help you get started with Pydantic AI Gateway, some code examples on the Pydantic AI documentation include a “Via Pydantic AI Gateway” tab, alongside a “Direct to Provider API” tab with the standard Pydantic AI model string. The main difference between them is that when using Gateway, model strings use the `gateway/` prefix.

- **API key management**: Access multiple LLM providers with a single Gateway key.
- **Cost Limits**: Set spending limits at project, user, and API key levels with daily, weekly, and monthly caps.
- **BYOK and managed providers:**Bring your own API keys (BYOK) from LLM providers, or pay for inference directly through the platform.
- **Multi-provider support:**Access models from OpenAI, Anthropic, Google Vertex, Groq, and AWS Bedrock.- *More providers coming soon*.
- **Routing groups:**Configure routing groups to fail over between providers serving the same model, or load-balance traffic across them by weight.
- **Backend observability:**Log every request through Pydantic Logfire or any OpenTelemetry backend (- *coming soon*).
- **Zero translation**: Unlike traditional AI gateways that translate everything to one common schema,- **Pydantic AI Gateway**allows requests to flow through directly in each provider’s native format. This gives you immediate access to new model features as soon as they are released.
- **Enterprise ready**: Inherits Logfire’s enterprise features — including SSO, custom roles and permissions.

This section contains instructions on how to set up your account and run your app with Pydantic AI Gateway credentials.

- Sign up at logfire.pydantic.dev
- Choose a region and create an account.
- Activate the gateway in your organizations settings.

Go to your organization’s Gateway settings in Logfire and create an API key.

After setting up your account with the instructions above, you will be able to make an AI model request with the Pydantic AI Gateway. The code snippets below show how you can use Pydantic AI Gateway with different frameworks and SDKs.

To use different models, change the model string `gateway/<api_format>:<model_name>` to other models offered by the supported providers.

Examples of providers and models that can be used are:

| Provider | API Format | Example Model | 
|---|---|---|
| OpenAI | `openai` | `gateway/openai:gpt-5.2` | 
| Anthropic | `anthropic` | `gateway/anthropic:claude-sonnet-4-6` | 
| Google Cloud (formerly Vertex AI) | `google-cloud` | `gateway/google-cloud:gemini-3-flash-preview` | 
| Groq | `groq` | `gateway/groq:openai/gpt-oss-120b` | 
| AWS Bedrock | `bedrock` | `gateway/bedrock:amazon.nova-micro-v1:0` | 

Before you start, make sure you are on version 1.16 or later of `pydantic-ai`. To update to the latest version run:

Set the `PYDANTIC_AI_GATEWAY_API_KEY` environment variable to your Gateway API key:

You can access multiple models with the same API key, as shown in the code snippet below.

Pass your API key directly using the `gateway_provider`:

To use an alternate provider or routing group, you can specify it in the route parameter:

Before you start, log out of Claude Code using `/logout`.

Set your gateway credentials as environment variables, using the base URL that matches your Logfire region:

Replace `YOUR_GATEWAY_API_KEY` with the API key from your Logfire organization’s Gateway settings.

Launch Claude Code by typing `claude`. All requests will now route through the Pydantic AI Gateway.

Codex uses the OpenAI Responses API, so it should use the Gateway’s `openai-responses` route.

Set your gateway API key as an environment variable:

Then add the following configuration to `~/.codex/config.toml`, using the base URL that matches your Logfire region:

```
model = "gpt-5.4"
model_provider = "pydantic_gateway"
[model_providers.pydantic_gateway]
name = "Pydantic AI Gateway"
base_url = "https://gateway-us.pydantic.dev/proxy/openai-responses"
env_key = "PYDANTIC_AI_GATEWAY_API_KEY"
env_key_instructions = "Create a Gateway API key in your Logfire organization's Gateway settings."
wire_api = "responses"
```
```
model = "gpt-5.4"
model_provider = "pydantic_gateway"
[model_providers.pydantic_gateway]
name = "Pydantic AI Gateway"
base_url = "https://gateway-eu.pydantic.dev/proxy/openai-responses"
env_key = "PYDANTIC_AI_GATEWAY_API_KEY"
env_key_instructions = "Create a Gateway API key in your Logfire organization's Gateway settings."
wire_api = "responses"
```
For more details on configuring custom providers in Codex, see the Codex custom model providers docs and the Codex configuration reference.

If you already have a `~/.codex/config.toml`, add the `[model_providers.pydantic_gateway]` block and update `model_provider` instead of replacing the whole file. Replace `gpt-5.4` with whichever OpenAI Responses model you want Codex to use.

Launch Codex by typing `codex`. All requests will now route through the Pydantic AI Gateway.

Use the base URL that matches your Logfire region (`gateway-us` or `gateway-eu`).

Use the base URL that matches your Logfire region (`gateway-us` or `gateway-eu`).

The Vercel AI SDK can route through the Gateway by pointing each provider’s `baseURL` at the matching proxy path (e.g. `/proxy/openai` or `/proxy/anthropic`). Use the base URL that matches your Logfire region (`gateway-us` or `gateway-eu`).

```
import { createOpenAI } from "@ai-sdk/openai";
import { generateText } from "ai";
const apiKey = process.env.PYDANTIC_AI_GATEWAY_API_KEY;
if (!apiKey) throw new Error("set PYDANTIC_AI_GATEWAY_API_KEY");
const openai = createOpenAI({
  apiKey,
  baseURL: "https://gateway-us.pydantic.dev/proxy/openai",
});
async function main() {
  const openaiResult = await generateText({
    model: openai("gpt-5.2"),
    prompt: "what color is the sky? reply concisely",
  });
  console.log("openai:", openaiResult.text);
}
main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```
```
import { createOpenAI } from "@ai-sdk/openai";
import { generateText } from "ai";
const apiKey = process.env.PYDANTIC_AI_GATEWAY_API_KEY;
if (!apiKey) throw new Error("set PYDANTIC_AI_GATEWAY_API_KEY");
const openai = createOpenAI({
  apiKey,
  baseURL: "https://gateway-eu.pydantic.dev/proxy/openai",
});
async function main() {
  const openaiResult = await generateText({
    model: openai("gpt-5.2"),
    prompt: "what color is the sky? reply concisely",
  });
  console.log("openai:", openaiResult.text);
}
main().catch((err) => {
  console.error(err);
  process.exit(1);
});
```
A **routing group** is a named collection of providers that all serve the same model. Each member has a **priority**, a **weight**, and an **active** flag, and those three values together let a single group express two different routing strategies:

- **Failover / fallback**: Assign members different priorities. The Gateway always tries the highest-priority active member first, and only falls through to a lower-priority member when the higher one is unavailable (for example if it is down, rate-limited, or returns an error).
- **Load balancing**: Assign two or more members the same priority and give each a weight. The Gateway splits traffic across those members in proportion to their weights.

The two strategies compose: you can have, for example, a top priority tier with two providers load-balanced 70/30, and a second priority tier that only receives traffic when both top-tier providers fail.

Routing groups are managed from your organization’s Gateway settings in Logfire:

- Open **Gateway -> Routing Groups**and click**Add Routing Group**.
- Give the group a slug (e.g. `anthropic-routing`) and an optional description.
- Open the group’s **Members**page and add one or more providers. For each member set:- **Priority**- higher values are tried first. Use different priorities across members for failover.
- **Weight**- load-balancing weight used between members that share the same priority.
- **Active**- inactive members are skipped during routing.
 

Point the Gateway provider at the group via the `route` parameter (the group’s slug):

The slug of the routing group you created in Logfire.

The gateway needs to know the cost of a request in order to provide spend insights and enforce spending limits.

Each provider has a **Require pricing data** toggle in its settings. When enabled (the default), the gateway rejects requests for models it has no pricing data for before forwarding them upstream. When disabled, those requests are allowed through, but their cost will not be tracked and they will not count toward spending limits.

The rejection response depends on the provider type:

- **Built-in providers**(Pydantic-managed):- `404`with a message asking you to let us know on Slack so we can add the model.
- **Custom providers**(your own API keys):- `400`indicating that pricing data is required, with a hint to disable the toggle if you want the request through anyway.

We are actively working on supporting more providers and models. If there’s a specific provider or model you’d like to see supported, please let us know on Slack or open an issue on `genai-prices`.

# Citations

1. Source page: https://pydantic.dev/docs/ai/overview/gateway
