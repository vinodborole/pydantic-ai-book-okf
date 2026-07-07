---
type: Web Page
title: Native Tools | Pydantic Docs
resource: https://pydantic.dev/docs/ai/tools-toolsets/native-tools
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Native Tools

Native tools are native tools provided by LLM providers that can be used to enhance your agent’s capabilities. Unlike common tools, which are custom implementations that Pydantic AI executes, native tools are executed directly by the model provider.

Pydantic AI supports the following native tools:

- `WebSearchTool`
- `XSearchTool`
- `CodeExecutionTool`
- `ImageGenerationTool`
- `WebFetchTool`
- `MemoryTool`
- `MCPServerTool`
- `FileSearchTool`

These tools are passed to the agent’s `capabilities` list, wrapped in `NativeTool`, and are executed by the model provider’s infrastructure.

Sometimes you need to configure a native tool dynamically based on the run context (e.g., user dependencies), or conditionally omit it. You can achieve this by wrapping a function with `NativeTool` in `capabilities`. The function takes `RunContext` as an argument and returns an `AbstractNativeTool` or `None`.

This is particularly useful for tools like `WebSearchTool` where you might want to set the user’s location based on the current request, or disable the tool if the user provides no location.

The `WebSearchTool` allows your agent to search the web,
making it ideal for queries that require up-to-date data.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI Responses | ✅ | Full feature support. To include search results on the `NativeToolReturnPart`that’s available via`ModelResponse.native_tool_calls`, enable the`OpenAIResponsesModelSettings.openai_include_web_search_sources`model setting. | 
| Anthropic | ✅ | Full feature support | 
| ✅ | No parameter support. No `NativeToolCallPart`or`NativeToolReturnPart`is generated when streaming. Using native tools and function tools (including output tools) at the same time is not supported; to use structured output, use`PromptedOutput`instead. | |
| xAI | ✅ | Supports `blocked_domains`,`allowed_domains`, and`user_location`parameters. | 
| Groq | ✅ | Limited parameter support. To use web search capabilities with Groq, you need to use the compound models. | 
| OpenRouter | ✅ | Web search via plugins. Supports `search_context_size`. Uses native search for supported providers (OpenAI, Anthropic, Perplexity, xAI), Exa for others. | 
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
| `user_location` | ✅ | ✅ | ✅ | ❌ | ❌ | 
| `blocked_domains` | ❌ | ✅ | ✅ | ✅ | ❌ | 
| `allowed_domains` | ✅ | ✅ | ✅ | ✅ | ❌ | 
| `max_uses` | ❌ | ✅ | ❌ | ❌ | ❌ | 

The `XSearchTool` allows your agent to search X/Twitter for real-time posts and content. Natively supported by xAI models; usable on other models via the `XSearch` capability with `fallback_model` set. See the xAI X Search documentation for more details.

*(This example is complete, it can be run “as is”)*

The `XSearchTool` supports several configuration parameters:

*(This example is complete, it can be run “as is”)*

The `CodeExecutionTool` enables your agent to execute code
in a secure environment, making it perfect for computational tasks, data analysis, and mathematical operations.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI | ✅ | To include code execution output on the `NativeToolReturnPart`that’s available via`ModelResponse.native_tool_calls`, enable the`OpenAIResponsesModelSettings.openai_include_code_execution_outputs`model setting. If the code execution generated images, like charts, they will be available on`ModelResponse.images`as`BinaryImage`objects. The generated image can also be used as image output for the agent run. | 
| ✅ | Using native tools and function tools (including output tools) at the same time is not supported; to use structured output, use `PromptedOutput`instead. | |
| Anthropic | ✅ | Available on compatible Anthropic models. Pydantic AI selects a compatible code execution tool version automatically; see Anthropic code execution tool version to override it. | 
| xAI | ✅ | Full feature support. | 
| Groq | ❌ | |
| Bedrock | ✅ | Only available for Nova 2.0 models. | 
| Mistral | ❌ | |
| Cohere | ❌ | |
| HuggingFace | ❌ | 

*(This example is complete, it can be run “as is”)*

In addition to text output, code execution with OpenAI can generate images as part of their response. Accessing this image via `ModelResponse.images` or image output requires the `OpenAIResponsesModelSettings.openai_include_code_execution_outputs` model setting to be enabled.

*(This example is complete, it can be run “as is”)*

The `ImageGenerationTool` enables your agent to generate images.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI Responses | ✅ | Full feature support. Only supported by models newer than `gpt-5.2`. Metadata about the generated image, like the`revised_prompt`sent to the underlying image model, is available on the`NativeToolReturnPart`that’s available via`ModelResponse.native_tool_calls`. | 
| ✅ | Limited parameter support. Only supported by image generation models like `gemini-3-pro-image-preview`and`gemini-3-pro-image-preview`. These models do not support function tools and will always have the option of generating images, even if this native tool is not explicitly specified. | |
| Anthropic | ❌ | |
| xAI | ❌ | |
| Groq | ❌ | |
| Bedrock | ❌ | |
| Mistral | ❌ | |
| Cohere | ❌ | |
| HuggingFace | ❌ | 

Generated images are available on `ModelResponse.images` as `BinaryImage` objects:

*(This example is complete, it can be run “as is”)*

Image generation with Google image generation models does not require the `ImageGenerationTool` native tool to be explicitly specified:

*(This example is complete, it can be run “as is”)*

The `ImageGenerationTool` can be used together with `output_type=BinaryImage` to get image output. If the `ImageGenerationTool` native tool is not explicitly specified, it will be enabled automatically:

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

For more details, check the API documentation.

| Parameter | OpenAI | |
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

The `WebFetchTool` enables your agent to pull URL contents into its context,
allowing it to pull up-to-date information from the web.

| Provider | Supported | Notes | 
|---|---|---|
| Anthropic | ✅ | Full feature support. Uses Anthropic’s Web Fetch Tool internally to retrieve URL contents. | 
| ✅ | No parameter support. The limits are fixed at 20 URLs per request with a maximum of 34MB per URL. Using native tools and function tools (including output tools) at the same time is not supported; to use structured output, use `PromptedOutput`instead. | |
| xAI | ❌ | Web browsing is implemented as part of `WebSearchTool`with xAI. | 
| OpenAI | ❌ | |
| Groq | ❌ | |
| Bedrock | ❌ | |
| Mistral | ❌ | |
| Cohere | ❌ | |
| HuggingFace | ❌ | 

*(This example is complete, it can be run “as is”)*

The `WebFetchTool` supports several configuration parameters:

*(This example is complete, it can be run “as is”)*

| Parameter | Anthropic | |
|---|---|---|
| `max_uses` | ✅ | ❌ | 
| `allowed_domains` | ✅ | ❌ | 
| `blocked_domains` | ✅ | ❌ | 
| `enable_citations` | ✅ | ❌ | 
| `max_content_tokens` | ✅ | ❌ | 

The `MemoryTool` enables your agent to use memory.

| Provider | Supported | Notes | 
|---|---|---|
| Anthropic | ✅ | Requires a tool named `memory`to be defined that implements specific sub-commands. You can use a subclass of`anthropic.lib.tools.BetaAbstractMemoryTool`as documented below. | 
| ❌ | ||
| OpenAI | ❌ | |
| Groq | ❌ | |
| Bedrock | ❌ | |
| Mistral | ❌ | |
| Cohere | ❌ | |
| HuggingFace | ❌ | 

The Anthropic SDK provides an abstract `BetaAbstractMemoryTool` class that you can subclass to create your own memory storage solution (e.g., database, cloud storage, encrypted files, etc.). Their `LocalFilesystemMemoryTool` example can serve as a starting point.

The following example uses a subclass that hard-codes a specific memory. The bits specific to Pydantic AI are the `MemoryTool` native tool and the `memory` tool definition that forwards commands to the `call` method of the `BetaAbstractMemoryTool` subclass.

*(This example is complete, it can be run “as is”)*

The `MCPServerTool` allows your agent to use remote MCP servers with communication handled by the model provider.

This requires the MCP server to live at a public URL the provider can reach and does not support many of the advanced features of Pydantic AI’s agent-side MCP support, but can result in optimized context use and caching, and faster performance due to the lack of a round-trip back to Pydantic AI.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI Responses | ✅ | Full feature support. Connectors can be used by specifying a special `x-openai-connector:<connector_id>`URL. | 
| Anthropic | ✅ | Full feature support | 
| xAI | ✅ | Full feature support | 
| ❌ | Not supported | |
| Groq | ❌ | Not supported | 
| OpenAI Chat Completions | ❌ | Not supported | 
| Bedrock | ❌ | Not supported | 
| Mistral | ❌ | Not supported | 
| Cohere | ❌ | Not supported | 
| HuggingFace | ❌ | Not supported | 

- The DeepWiki MCP server does not require authorization.

*(This example is complete, it can be run “as is”)*

With OpenAI, you must use their Responses API to access the MCP server tool:

- The DeepWiki MCP server does not require authorization.

*(This example is complete, it can be run “as is”)*

The `MCPServerTool` supports several configuration parameters for custom MCP servers:

- The GitHub MCP server requires an authorization token.

For OpenAI Responses, you can use a connector by specifying a special `x-openai-connector:` URL:

*(This example is complete, it can be run “as is”)*

- OpenAI’s Google Calendar connector requires an authorization token.

*(This example is complete, it can be run “as is”)*

| Parameter | OpenAI | Anthropic | xAI | 
|---|---|---|---|
| `authorization_token` | ✅ | ✅ | ✅ | 
| `allowed_tools` | ✅ | ✅ | ✅ | 
| `description` | ✅ | ❌ | ✅ | 
| `headers` | ✅ | ❌ | ✅ | 

The `FileSearchTool` enables your agent to search through uploaded files using vector search, providing a fully managed Retrieval-Augmented Generation (RAG) system. This tool handles file storage, chunking, embedding generation, and context injection into prompts.

| Provider | Supported | Notes | 
|---|---|---|
| OpenAI Responses | ✅ | Full feature support. Requires files to be uploaded to vector stores via the OpenAI Files API. To include search results on the `NativeToolReturnPart`available via`ModelResponse.native_tool_calls`, enable the`OpenAIResponsesModelSettings.openai_include_file_search_results`model setting. | 
| Google (Gemini) | ✅ | Requires files to be uploaded via the Gemini Files API. Files are automatically deleted after 48 hours. Supports up to 2 GB per file and 20 GB per project. Using native tools and function tools (including output tools) at the same time is not supported; to use structured output, use `PromptedOutput`instead. | 
| xAI | ✅ | Mapped to xAI collections search. Requires collection IDs. To include search results on the `NativeToolReturnPart`, enable the`XaiModelSettings.xai_include_collections_search_output`model setting. | 
| Google Cloud | ❌ | |
| Anthropic | ❌ | Not supported | 
| Groq | ❌ | Not supported | 
| OpenAI Chat Completions | ❌ | Not supported | 
| Bedrock | ❌ | Not supported | 
| Mistral | ❌ | Not supported | 
| Cohere | ❌ | Not supported | 
| HuggingFace | ❌ | Not supported | 

With OpenAI, you need to first upload files to a vector store, then reference the vector store IDs when using the `FileSearchTool`.

With Gemini, you need to first create a file search store via the Files API, then reference the file search store names.

With xAI, `FileSearchTool` maps to the collections search tool. Pass collection IDs as `file_store_ids`.

For complete API documentation, see the API Reference.

# Citations

1. Source page: https://pydantic.dev/docs/ai/tools-toolsets/native-tools
