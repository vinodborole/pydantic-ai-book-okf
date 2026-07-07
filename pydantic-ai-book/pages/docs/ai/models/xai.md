---
type: Web Page
title: xAI | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/xai
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# xAI

To use `XaiModel`, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `xai` optional group:

To use xAI models from xAI through their API, go to console.x.ai to create an API key.

docs.x.ai contains a list of available xAI models.

Once you have the API key, you can set it as an environment variable:

You can then use `XaiModel` by name:

```
from pydantic_ai import Agent
agent = Agent('xai:grok-4.3')
...
```
Or initialise the model directly:

```
from pydantic_ai import Agent
from pydantic_ai.models.xai import XaiModel
# Uses XAI_API_KEY environment variable
model = XaiModel('grok-4.3')
agent = Agent(model)
...
```
You can also customize the `XaiModel` with a custom provider:

```
from pydantic_ai import Agent
from pydantic_ai.models.xai import XaiModel
from pydantic_ai.providers.xai import XaiProvider
# Custom API key
provider = XaiProvider(api_key='your-api-key')
model = XaiModel('grok-4.3', provider=provider)
agent = Agent(model)
...
```
For gateway, regional, or proxy deployments you can also point the provider at a custom host and set a client-level default timeout, both of which are forwarded to the underlying `xai_sdk.AsyncClient`:

```
from pydantic_ai import Agent
from pydantic_ai.models.xai import XaiModel
from pydantic_ai.providers.xai import XaiProvider
provider = XaiProvider(
    api_key='your-api-key',
    api_host='gateway.example.com',
    timeout=30,
)
model = XaiModel('grok-4.3', provider=provider)
agent = Agent(model)
...
```
`api_host` is the hostname of the xAI API server (the SDK connects over gRPC), and `timeout` is the default timeout in seconds applied to every request the client makes. The provider-level `timeout` is distinct from `ModelSettings.timeout`, which overrides the timeout for an individual request. Both options are omitted when left unset, so the SDK’s own defaults apply.

Or with a custom `xai_sdk.AsyncClient`:

```
from xai_sdk import AsyncClient
from pydantic_ai import Agent
from pydantic_ai.models.xai import XaiModel
from pydantic_ai.providers.xai import XaiProvider
xai_client = AsyncClient(api_key='your-api-key')
provider = XaiProvider(xai_client=xai_client)
model = XaiModel('grok-4.3', provider=provider)
agent = Agent(model)
...
```
xAI models support searching X (formerly Twitter) for real-time posts and content. The recommended way to enable it is with the `XSearch` capability — see the capability documentation for more details, including cross-provider usage. For the full list of supported options, see the xAI X Search documentation.

*(This example is complete, it can be run “as is”)*

The `XSearch` capability accepts:

- `allowed_x_handles`- `excluded_x_handles`
- `from_date`- `to_date`
- `enable_image_understanding`- `False`): analyze images attached to posts.
- `enable_video_understanding`- `False`): analyze video content attached to posts.
- `include_output`- `False`): include the raw X search results on the- `NativeToolReturnPart`available via- `ModelResponse.native_tool_calls`. Without this, the model uses the search results internally but only returns its text summary; enabling it gives programmatic access to the searched posts, sources, and metadata.

As an alternative to the capability, you can pass the lower-level `XSearchTool` directly via `capabilities=[NativeTool(XSearchTool(...))]` — see the X Search Tool documentation — or enable raw output globally via the `XaiModelSettings.xai_include_x_search_output` model setting.

Grok 4.3 supports `reasoning_effort` values of `'none'`, `'low'`, `'medium'`, and `'high'`. You can configure it directly with `XaiModelSettings.xai_reasoning_effort`, or use the cross-provider `ModelSettings.thinking` setting:

Set `xai_reasoning_effort='none'` or `thinking=False` to disable reasoning on Grok 4.3. xAI redirects several retired text model slugs to `grok-4.3`; choose `grok-4.3` and an explicit reasoning effort when you need predictable behavior and cost. See the xAI May 15 retirement guide for details.

When a request uses xAI’s server-side native tools (e.g. web search, code execution, X search), xAI runs its own loop — calling those tools and processing their results — before returning a final response. You can cap how many turns that server-side loop may take with `XaiModelSettings.xai_max_turns`:

`xai_max_turns` only governs xAI’s server-side native-tool loop. It has no effect on ordinary client-side tools or on Pydantic AI’s own agent loop — to bound those, use `UsageLimits`.

Note that when parallel tool calls are enabled, multiple tool calls can occur within a single turn, so `xai_max_turns` does not necessarily equal the total number of tool calls made.

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/xai
