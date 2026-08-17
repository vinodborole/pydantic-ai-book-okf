---
type: Web Page
title: Native Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/native-tools
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Native Tools

Native tools are native tools provided by LLM providers that can be used to enhance your agent’s capabilities. Unlike [common tools](/docs/ai/tools-toolsets/common-tools/), which are custom implementations that Pydantic AI executes, native tools are executed directly by the model provider.

Pydantic AI supports the following native tools:

- **[`WebSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebSearchTool)** : Allows agents to search the web
- **[`XSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool)** : Allows agents to search X/Twitter (xAI only)
- **[`CodeExecutionTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.CodeExecutionTool)** : Enables agents to execute code in a secure environment
- **[`ImageGenerationTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.ImageGenerationTool)** : Enables agents to generate images
- **[`WebFetchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebFetchTool)** : Enables agents to fetch web pages
- **[`MemoryTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.MemoryTool)** : Enables agents to use memory
- **[`MCPServerTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.MCPServerTool)** : Enables agents to use remote MCP servers with communication handled by the model provider
- **[`FileSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.FileSearchTool)** : Enables agents to search through uploaded files using vector search (RAG)
- **[`AdvisorTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.AdvisorTool)** : Lets a faster executor model consult a stronger advisor model mid-generation (Anthropic, OpenRouter)

These tools are passed to the agent’s `capabilities` list, wrapped in [`NativeTool`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.NativeTool), and are executed by the model provider’s infrastructure.

[Gemini 3 models](https://ai.google.dev/gemini-api/docs/structured-output#structured_outputs_with_tools) support combining native tools with function tools, including [output tools](/docs/ai/core-concepts/output/#tool-output), and [`NativeOutput`](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.NativeOutput). Earlier Gemini models cannot use these combinations; use [`PromptedOutput`](/docs/ai/api/pydantic-ai/output/#pydantic_ai.output.PromptedOutput) for structured output alongside native tools.

Sometimes you need to configure a native tool dynamically based on the [run context](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) (e.g., user dependencies), or conditionally omit it. You can achieve this by wrapping a function with [`NativeTool`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.NativeTool) in `capabilities`. The function takes [`RunContext`](/docs/ai/api/pydantic-ai/tools/#pydantic_ai.tools.RunContext) as an argument and returns an [`AbstractNativeTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.AbstractNativeTool) or `None`.

This is particularly useful for tools like [`WebSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebSearchTool) where you might want to set the user’s location based on the current request, or disable the tool if the user provides no location.

The [`WebSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebSearchTool) allows your agent to search the web,
making it ideal for queries that require up-to-date data.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI Responses | ✅ | Full feature support. To include search results on the [`NativeToolReturnPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.NativeToolReturnPart) that’s available via[`ModelResponse.native_tool_calls`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse.native_tool_calls) , enable the[`OpenAIResponsesModelSettings.openai_include_web_search_sources`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIResponsesModelSettings.openai_include_web_search_sources)[model setting](/docs/ai/core-concepts/agent/#model-run-settings) . | 
| Anthropic | ✅ | Full feature support | 
|  | ✅ | No parameter support. No [`NativeToolCallPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.NativeToolCallPart) or[`NativeToolReturnPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.NativeToolReturnPart) is generated when streaming. See[Google tool combinations](#google-tool-combinations) . | 
| xAI | ✅ | Supports `blocked_domains` ,`allowed_domains` , and`user_location` parameters. | 
| Groq | ✅ | Limited parameter support. To use web search capabilities with Groq, you need to use the [compound models](https://console.groq.com/docs/compound) . | 
| OpenRouter | ✅ | Uses OpenRouter’s [Beta web-search server tool](https://openrouter.ai/docs/guides/features/server-tools/web-search) . The model can make 0–N searches. Recorded requests verify only that OpenRouter accepts the parameter names; the per-engine effects below are per OpenRouter’s docs: native search ignores`search_context_size` ;`user_location` is native-only; native OpenAI ignores`blocked_domains` ; and`max_uses` works with non-native or Anthropic native search. | 
| OpenAI Chat Completions | ❌ | Not supported | 
| Bedrock | ❌ | Not supported | 
| Mistral | ❌ | Not supported | 
| Cohere | ❌ | Not supported | 
| HuggingFace | ❌ | Not supported | 

*(This example is complete, it can be run “as is”)*

With OpenAI, you must use their Responses API to access the web search tool.

*(This example is complete, it can be run “as is”)*

The `WebSearchTool` supports several configuration parameters:

*(This example is complete, it can be run “as is”)*

| Parameter | OpenAI | Anthropic | xAI | Groq | OpenRouter | 
|---|---|---|---|---|---|
| `search_context_size` | ✅ | ❌ | ❌ | ❌ | ✅ | 
| `user_location` | ✅ | ✅ | ✅ | ❌ | ✅ | 
| `blocked_domains` | ❌ | ✅ | ✅ | ✅ | ✅ | 
| `allowed_domains` | ✅ | ✅ | ✅ | ✅ | ✅ | 
| `max_uses` | ❌ | ✅ | ❌ | ❌ | ✅* | 
| `external_web_access` | ✅ | ❌ | ❌ | ❌ | ❌ | 

- Per OpenRouter’s documentation, native provider search forwards `max_uses` only to Anthropic; other native providers ignore it.

The [`XSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.XSearchTool) allows your agent to search X/Twitter for real-time posts and content. Natively supported by xAI models; usable on other models via the [`XSearch`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.XSearch) capability with `fallback_model` set. See the [xAI X Search documentation](https://docs.x.ai/developers/tools/x-search) for more details.

*(This example is complete, it can be run “as is”)*

The `XSearchTool` supports several configuration parameters:

*(This example is complete, it can be run “as is”)*

The [`CodeExecutionTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.CodeExecutionTool) enables your agent to execute code
in a secure environment, making it perfect for computational tasks, data analysis, and mathematical operations.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI | ✅ | To include code execution output on the [`NativeToolReturnPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.NativeToolReturnPart) that’s available via[`ModelResponse.native_tool_calls`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse.native_tool_calls) , enable the[`OpenAIResponsesModelSettings.openai_include_code_execution_outputs`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIResponsesModelSettings.openai_include_code_execution_outputs)[model setting](/docs/ai/core-concepts/agent/#model-run-settings) . If the code execution generated images, like charts, they will be available on[`ModelResponse.images`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse.images) as[`BinaryImage`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryImage) objects. The generated image can also be used as[image output](/docs/ai/core-concepts/output/#image-output) for the agent run. | 
|  | ✅ | See [Google tool combinations](#google-tool-combinations) . | 
| Anthropic | ✅ | Available on compatible Anthropic models. Pydantic AI selects a compatible code execution tool version automatically; see [Anthropic code execution tool version](/docs/ai/models/anthropic/#code-execution-tool-version) to override it. | 
| xAI | ✅ | Full feature support. | 
| Groq | ❌ |  | 
| Bedrock | ✅ | Only available for Nova 2.0 models. | 
| Mistral | ❌ |  | 
| Cohere | ❌ |  | 
| HuggingFace | ❌ |  | 

*(This example is complete, it can be run “as is”)*

In addition to text output, code execution with OpenAI can generate images as part of their response. Accessing this image via [`ModelResponse.images`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse.images) or [image output](/docs/ai/core-concepts/output/#image-output) requires the `OpenAIResponsesModelSettings.openai_include_code_execution_outputs`[model setting](/docs/ai/core-concepts/agent/#model-run-settings) to be enabled.

*(This example is complete, it can be run “as is”)*

You can upload files via the provider’s Files API and make them available to the code execution container. This allows the agent to process data files, analyze CSVs, work with images, and more.
Files whose [`UploadedFile.provider_name`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.UploadedFile.provider_name) does not match the model provider are ignored.

For details on file management, persistence, and container behavior, see the [Anthropic Files API documentation](https://platform.claude.com/docs/en/build-with-claude/files).

For details on file management, container lifecycle, and persistence behavior, see the [OpenAI Responses API documentation](https://platform.openai.com/docs/api-reference/responses).

| Parameter | Anthropic | OpenAI |  | xAI | 
|---|---|---|---|---|
| `files` | ✅ | ✅ | ❌ | ❌ | 

The [`ImageGenerationTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.ImageGenerationTool) enables your agent to generate images.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI Responses | ✅ | Full feature support. Only supported by models newer than `gpt-5.2` . Metadata about the generated image, like the[`revised_prompt`](https://platform.openai.com/docs/guides/tools-image-generation#revised-prompt) sent to the underlying image model, is available on the[`NativeToolReturnPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.NativeToolReturnPart) that’s available via[`ModelResponse.native_tool_calls`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse.native_tool_calls) . | 
|  | ✅ | Limited parameter support. Only supported by [image generation models](https://ai.google.dev/gemini-api/docs/image-generation) like`gemini-3-pro-image` and`gemini-3.1-flash-image` . These models do not support[function tools](/docs/ai/tools-toolsets/tools/) and will always have the option of generating images, even if this native tool is not explicitly specified. | 
| Anthropic | ❌ |  | 
| xAI | ❌ |  | 
| Groq | ❌ |  | 
| Bedrock | ❌ |  | 
| Mistral | ❌ |  | 
| Cohere | ❌ |  | 
| HuggingFace | ❌ |  | 

Generated images are available on [`ModelResponse.images`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse.images) as [`BinaryImage`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryImage) objects:

*(This example is complete, it can be run “as is”)*

Image generation with Google [image generation models](https://ai.google.dev/gemini-api/docs/image-generation) does not require the `ImageGenerationTool` native tool to be explicitly specified:

*(This example is complete, it can be run “as is”)*

The `ImageGenerationTool` can be used together with `output_type=BinaryImage` to get [image output](/docs/ai/core-concepts/output/#image-output). If the `ImageGenerationTool` native tool is not explicitly specified, it will be enabled automatically:

*(This example is complete, it can be run “as is”)*

The `ImageGenerationTool` supports several configuration parameters:

*(This example is complete, it can be run “as is”)*

OpenAI Responses models also respect the `aspect_ratio` parameter. Because the OpenAI API only exposes discrete image sizes,
Pydantic AI maps `'1:1'` -> `1024x1024`, `'2:3'` -> `1024x1536`, and `'3:2'` -> `1536x1024`. Providing any other aspect ratio
results in an error, and if you also set `size` it must match the computed value.

The OpenAI Responses image generation tool defaults to `action='auto'`, where the model decides whether to generate a new
image or edit one already in context. Use `action='generate'` or `action='edit'` to force either behavior. You can also set
`model` to select the underlying image generation model used by the tool, for example `model='gpt-image-2'`; this does not
change the agent’s conversational model.

To control the aspect ratio when using Gemini image models, include the `ImageGenerationTool` explicitly:

*(This example is complete, it can be run “as is”)*

To control the image resolution with Google image generation models (starting with Gemini 3 Pro Image), use the `size` parameter:

*(This example is complete, it can be run “as is”)*

For more details, check the [API documentation](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.ImageGenerationTool).

| Parameter | OpenAI |  | 
|---|---|---|
| `action` | ✅ (auto (default), generate, edit) | ❌ | 
| `background` | ✅ | ❌ | 
| `input_fidelity` | ✅ | ❌ | 
| `moderation` | ✅ | ❌ | 
| `model` | ✅ (gpt-image-2, gpt-image-1.5, gpt-image-1, gpt-image-1-mini, or another OpenAI image model ID) | ❌ | 
| `output_compression` | ✅ (100 (default), jpeg or webp only) | ✅ (75 (default), jpeg only, Google Cloud only) | 
| `output_format` | ✅ | ✅ (Google Cloud only) | 
| `partial_images` | ✅ | ❌ | 
| `quality` | ✅ | ❌ | 
| `size` | ✅ (auto (default), 1024x1024, 1024x1536, 1536x1024) | ✅ (512, 1K (default), 2K, 4K) | 
| `aspect_ratio` | ✅ (1:1, 2:3, 3:2) | ✅ (1:1, 2:3, 3:2, 3:4, 4:3, 4:5, 5:4, 9:16, 16:9, 21:9) | 

The [`WebFetchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.WebFetchTool) enables your agent to pull URL contents into its context,
allowing it to pull up-to-date information from the web.

| Provider | Supported | Notes | 
|---|---|---|
| Anthropic | ✅ | Full feature support. Uses Anthropic’s [Web Fetch Tool](https://docs.claude.com/en/docs/agents-and-tools/tool-use/web-fetch-tool) internally to retrieve URL contents. | 
|  | ✅ | No parameter support. The limits are fixed at 20 URLs per request with a maximum of 34MB per URL. See [Google tool combinations](#google-tool-combinations) . | 
| xAI | ❌ | Web browsing is implemented as part of [`WebSearchTool`](#web-search-tool) with xAI. | 
| OpenAI | ❌ |  | 
| Groq | ❌ |  | 
| Bedrock | ❌ |  | 
| Mistral | ❌ |  | 
| Cohere | ❌ |  | 
| HuggingFace | ❌ |  | 

*(This example is complete, it can be run “as is”)*

The `WebFetchTool` supports several configuration parameters:

*(This example is complete, it can be run “as is”)*

| Parameter | Anthropic |  | 
|---|---|---|
| `max_uses` | ✅ | ❌ | 
| `allowed_domains` | ✅ | ❌ | 
| `blocked_domains` | ✅ | ❌ | 
| `enable_citations` | ✅ | ❌ | 
| `max_content_tokens` | ✅ | ❌ | 

The [`MemoryTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.MemoryTool) enables your agent to use memory.

| Provider | Supported | Notes | 
|---|---|---|
| Anthropic | ✅ | Requires a tool named `memory` to be defined that implements[specific sub-commands](https://docs.claude.com/en/docs/agents-and-tools/tool-use/memory-tool#tool-commands) . You can use a subclass of[`anthropic.lib.tools.BetaAbstractMemoryTool`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/lib/tools/_beta_builtin_memory_tool.py) as documented below. | 
|  | ❌ |  | 
| OpenAI | ❌ |  | 
| Groq | ❌ |  | 
| Bedrock | ❌ |  | 
| Mistral | ❌ |  | 
| Cohere | ❌ |  | 
| HuggingFace | ❌ |  | 

The Anthropic SDK provides an abstract [`BetaAbstractMemoryTool`](https://github.com/anthropics/anthropic-sdk-python/blob/main/src/anthropic/lib/tools/_beta_builtin_memory_tool.py) class that you can subclass to create your own memory storage solution (e.g., database, cloud storage, encrypted files, etc.). Their [`LocalFilesystemMemoryTool`](https://github.com/anthropics/anthropic-sdk-python/blob/main/examples/memory/basic.py) example can serve as a starting point.

The following example uses a subclass that hard-codes a specific memory. The bits specific to Pydantic AI are the `MemoryTool` native tool and the `memory` tool definition that forwards commands to the `call` method of the `BetaAbstractMemoryTool` subclass.

*(This example is complete, it can be run “as is”)*

The [`AdvisorTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.AdvisorTool) lets an executor model consult another model mid-generation. See the [Anthropic](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool) and [OpenRouter](https://openrouter.ai/docs/guides/features/server-tools/advisor) documentation for current model compatibility.

| Provider | Supported | Notes | 
|---|---|---|
| Anthropic | ✅ | Available on the Claude API and Claude Platform on AWS. | 
| OpenRouter | ✅ | Works with any executor model. | 
| OpenAI | ❌ |  | 
|  | ❌ |  | 
| xAI | ❌ |  | 
| Groq | ❌ |  | 
| Bedrock | ❌ |  | 
| Mistral | ❌ |  | 
| Cohere | ❌ |  | 
| HuggingFace | ❌ |  | 

For OpenRouter, use any `openrouter:` executor and pass an OpenRouter model slug to `model`, for example `anthropic/claude-opus-4.8`. Pydantic AI sends `forward_transcript=false`; `max_uses` and `caching` are ignored. Pydantic AI surfaces aggregate consultation counts under `ModelResponse.provider_details``['server_tool_use']`.

With Anthropic, Pydantic AI preserves plaintext and encrypted advisor results in message history, and strips advisor blocks when the tool is no longer enabled. Streaming pauses while the advisor runs. Advisor usage is reported under `advisor_*` keys in `RequestUsage.details` and excluded from the executor’s top-level token totals.

| Parameter | Anthropic | OpenRouter | 
|---|---|---|
| `model` | ✅ (required — the advisor model to consult) | ✅ (required — an OpenRouter catalog slug) | 
| `max_uses` | ✅ (cap on advisor consultations per request) | ❌ (fixed gateway limit; ignored) | 
| `max_tokens` | ✅ (cap on advisor output tokens, minimum 1024; makes the result carry a `stop_reason` ) | ✅ (maps to `max_completion_tokens` ) | 
| `caching` | ✅ ( `'5m'` or`'1h'` — ephemeral caching of the advisor context) | ❌ (no equivalent; ignored) | 

The [`MCPServerTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.MCPServerTool) allows your agent to use remote MCP servers with communication handled by the model provider.

This requires the MCP server to live at a public URL the provider can reach and does not support many of the advanced features of Pydantic AI’s agent-side [MCP support](/docs/ai/mcp/client/),
but can result in optimized context use and caching, and faster performance due to the lack of a round-trip back to Pydantic AI.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI Responses | ✅ | Full feature support. [Connectors](https://platform.openai.com/docs/guides/tools-connectors-mcp#connectors) can be used by specifying a special`x-openai-connector:<connector_id>` URL. | 
| Anthropic | ✅ | Full feature support | 
| xAI | ✅ | Full feature support | 
|  | ❌ | Not supported | 
| Groq | ❌ | Not supported | 
| OpenAI Chat Completions | ❌ | Not supported | 
| Bedrock | ❌ | Not supported | 
| Mistral | ❌ | Not supported | 
| Cohere | ❌ | Not supported | 
| HuggingFace | ❌ | Not supported | 

1. The [DeepWiki MCP server](https://docs.devin.ai/work-with-devin/deepwiki-mcp) does not require authorization.

*(This example is complete, it can be run “as is”)*

With OpenAI, you must use their Responses API to access the MCP server tool:

1. The [DeepWiki MCP server](https://docs.devin.ai/work-with-devin/deepwiki-mcp) does not require authorization.

*(This example is complete, it can be run “as is”)*

The `MCPServerTool` supports several configuration parameters for custom MCP servers:

1. The [GitHub MCP server](https://github.com/github/github-mcp-server) requires an authorization token.

For OpenAI Responses, you can use a [connector](https://platform.openai.com/docs/guides/tools-connectors-mcp#connectors) by specifying a special `x-openai-connector:` URL:

*(This example is complete, it can be run “as is”)*

1. OpenAI’s Google Calendar connector requires an [authorization token](https://platform.openai.com/docs/guides/tools-connectors-mcp#authorizing-a-connector) .

*(This example is complete, it can be run “as is”)*

| Parameter | OpenAI | Anthropic | xAI | 
|---|---|---|---|
| `authorization_token` | ✅ | ✅ | ✅ | 
| `allowed_tools` | ✅ | ✅ | ✅ | 
| `description` | ✅ | ❌ | ✅ | 
| `headers` | ✅ | ❌ | ✅ | 

The [`FileSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.FileSearchTool) enables your agent to search through uploaded files using vector search, providing a fully managed Retrieval-Augmented Generation (RAG) system. This tool handles file storage, chunking, embedding generation, and context injection into prompts.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI Responses | ✅ | Full feature support. Requires files to be uploaded to vector stores via the [OpenAI Files API](https://platform.openai.com/docs/api-reference/files) . To include search results on the[`NativeToolReturnPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.NativeToolReturnPart) available via[`ModelResponse.native_tool_calls`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ModelResponse.native_tool_calls) , enable the[`OpenAIResponsesModelSettings.openai_include_file_search_results`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIResponsesModelSettings.openai_include_file_search_results)[model setting](/docs/ai/core-concepts/agent/#model-run-settings) . | 
| Google (Gemini) | ✅ | Requires files to be uploaded via the [Gemini Files API](https://ai.google.dev/gemini-api/docs/files) . Files are automatically deleted after 48 hours. Supports up to 2 GB per file and 20 GB per project. See[Google tool combinations](#google-tool-combinations) . | 
| xAI | ✅ | Mapped to xAI collections search. Requires collection IDs. To include search results on the [`NativeToolReturnPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.NativeToolReturnPart) , enable the[`XaiModelSettings.xai_include_collections_search_output`](/docs/ai/api/models/xai/#pydantic_ai.models.xai.XaiModelSettings.xai_include_collections_search_output)[model setting](/docs/ai/core-concepts/agent/#model-run-settings) . | 
|  | Google Cloud | ❌ | 
| Anthropic | ❌ | Not supported | 
| Groq | ❌ | Not supported | 
| OpenAI Chat Completions | ❌ | Not supported | 
| Bedrock | ❌ | Not supported | 
| Mistral | ❌ | Not supported | 
| Cohere | ❌ | Not supported | 
| HuggingFace | ❌ | Not supported | 

With OpenAI, you need to first [upload files to a vector store](https://platform.openai.com/docs/assistants/tools/file-search), then reference the vector store IDs when using the `FileSearchTool`.

With Gemini, you need to first [create a file search store via the Files API](https://ai.google.dev/gemini-api/docs/files), then reference the file search store names.

With xAI, `FileSearchTool` maps to the [collections search](https://docs.x.ai/developers/tools/collections-search) tool. Pass collection IDs as `file_store_ids`.

xAI’s collections search also accepts options to control result count, ranking guidance, and retrieval strategy. These map to the `max_num_results`, `instructions`, and `retrieval_mode` fields on [`FileSearchTool`](/docs/ai/api/pydantic-ai/native_tools/#pydantic_ai.native_tools.FileSearchTool). When omitted, the server applies its own defaults (10 results, hybrid retrieval).

For complete API documentation, see the [API Reference](/docs/ai/api/pydantic-ai/native_tools/).

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/native-tools
