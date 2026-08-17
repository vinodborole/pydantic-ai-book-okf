---
type: Web Page
title: Multimodal Input | Pydantic Docs
resource: https://pydantic.dev/docs/ai/core-concepts/input
timestamp: '2026-08-17T07:03:21.217446+00:00'
---

# Multimodal Input

Alongside text, agents can accept image, audio, video, and document input, as long as the model supports it.

If you have a direct URL for the image, you can use [`ImageUrl`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ImageUrl):

If you have the image locally, you can also use [`BinaryContent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryContent):

To ensure the example is runnable we download this image from the web, but you can also use `Path().read_bytes()` to read a local file's contents.

You can provide audio input using either [`AudioUrl`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.AudioUrl) or [`BinaryContent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryContent). The process is analogous to the examples above.

You can provide video input using either [`VideoUrl`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.VideoUrl) or [`BinaryContent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryContent). The process is analogous to the examples above.

You can provide document input using either [`DocumentUrl`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.DocumentUrl) or [`BinaryContent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryContent). The process is similar to the examples above.

If you have a direct URL for the document, you can use [`DocumentUrl`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.DocumentUrl):

The supported document formats vary by model.

You can also use [`BinaryContent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.BinaryContent) to pass document data directly:

You can use [`TextContent`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.TextContent) to provide text input with additional metadata:

This is equivalent to passing the text as a `str`, but allows you to include additional `metadata` that can be accessed
programmatically in your agent logic.

When using one of `ImageUrl`, `AudioUrl`, `VideoUrl` or `DocumentUrl`, Pydantic AI will default to sending the URL to the model provider, so the file is downloaded on their side.

Support for file URLs varies depending on type and provider:

| Model | Send URL directly | Download and send bytes | Unsupported | 
|---|---|---|---|
| [`OpenAIChatModel`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIChatModel) | `ImageUrl` | `AudioUrl` ,`DocumentUrl` | `VideoUrl` .`DocumentUrl`[not supported with `AzureProvider`](/docs/ai/models/openai/#using-azure-with-the-responses-api) or[`AlibabaProvider`](/docs/ai/models/openai/#alibaba-cloud-model-studio-dashscope) | 
| [`OpenAIResponsesModel`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIResponsesModel) | `ImageUrl` ,`AudioUrl` ,`DocumentUrl` | — | `VideoUrl` | 
| [`AnthropicModel`](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModel) | `ImageUrl` ,`DocumentUrl` (PDF) | `DocumentUrl` (`text/plain` ) | `AudioUrl` ,`VideoUrl` | 
| [`GoogleModel`](/docs/ai/api/models/google/#pydantic_ai.models.google.GoogleModel) (Google Cloud) | All URL types | — | — | 
| [`GoogleModel`](/docs/ai/api/models/google/#pydantic_ai.models.google.GoogleModel) (Gemini API) | [YouTube](/docs/ai/models/google/#document-image-audio-and-video-input) ,[Files API](/docs/ai/models/google/#document-image-audio-and-video-input) | All other URLs | — | 
| [`XaiModel`](/docs/ai/api/models/xai/#pydantic_ai.models.xai.XaiModel) | `ImageUrl` | `DocumentUrl` | `AudioUrl` ,`VideoUrl` | 
| [`MistralModel`](/docs/ai/api/models/mistral/#pydantic_ai.models.mistral.MistralModel) | `ImageUrl` ,`DocumentUrl` (PDF) | `DocumentUrl` (`text/plain` ) | `AudioUrl` ,`VideoUrl` ,`DocumentUrl` (non-PDF, non-text) | 
| [`BedrockConverseModel`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockConverseModel) | S3 URLs ( `s3://` ) | `ImageUrl` ,`DocumentUrl` ,`VideoUrl` | `AudioUrl` | 
| [`OpenRouterModel`](/docs/ai/api/models/openrouter/#pydantic_ai.models.openrouter.OpenRouterModel) | `ImageUrl` ,`DocumentUrl` ,`VideoUrl` | `AudioUrl` | — | 

A model API may be unable to download a file (e.g., because of crawling or access restrictions) even if it supports file URLs. For example, [`GoogleModel`](/docs/ai/api/models/google/#pydantic_ai.models.google.GoogleModel) on Google Cloud limits YouTube video URLs to one URL per request.

In such cases, you can instruct Pydantic AI to download the file content locally and send that instead of the URL by setting `force_download` on the URL object:

Some model providers have their own file storage APIs where you can upload files and reference them by ID or URL.

Use [`UploadedFile`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.UploadedFile) to reference files that have been uploaded to a provider’s file storage API.

| Model | Support | 
|---|---|
| [`AnthropicModel`](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModel) | ✅ via [Anthropic Files API](https://docs.anthropic.com/en/docs/build-with-claude/files) | 
| [`OpenAIChatModel`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIChatModel) | ✅ via [OpenAI Files API](https://platform.openai.com/docs/api-reference/files) | 
| [`OpenAIResponsesModel`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIResponsesModel) | ✅ via [OpenAI Files API](https://platform.openai.com/docs/api-reference/files) | 
| [`GoogleModel`](/docs/ai/api/models/google/#pydantic_ai.models.google.GoogleModel) | ✅ via [Google Files API](https://ai.google.dev/gemini-api/docs/files) | 
| [`BedrockConverseModel`](/docs/ai/api/models/bedrock/#pydantic_ai.models.bedrock.BedrockConverseModel) | ✅ via S3 URLs ( `s3://bucket/key` ) | 
| [`XaiModel`](/docs/ai/api/models/xai/#pydantic_ai.models.xai.XaiModel) | ✅ via [xAI Files API](https://docs.x.ai/docs/guides/files) | 
| Other models | ❌ Not supported | 

When using [`UploadedFile`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.UploadedFile) you must set the `provider_name`. Uploaded files are specific to the system they are uploaded to and are not transferable across providers. Trying to use a message that contains an `UploadedFile` with a different provider will result in an error.

If you want to introduce portability into your agent logic to allow the same prompt history to work with different provider backends, you can use a [history processor](/docs/ai/core-concepts/message-history/#processing-message-history) to remove or rewrite `UploadedFile` parts from messages before sending them to a provider that does not support them. Be aware that stripping out `UploadedFile` instances might confuse the model, especially if references to those files remain in the text.

The `media_type` parameter is optional for [`UploadedFile`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.UploadedFile). If not specified, Pydantic AI will attempt to infer it from the `file_id`:

1. If `file_id` is a URL or path with a recognizable file extension (e.g.,`.pdf` ,`.png` ), the media type is inferred automatically
2. For opaque file IDs (e.g., `'file-abc123'` ), the media type defaults to`'application/octet-stream'`

Follow the [Anthropic Files API docs](https://docs.anthropic.com/en/docs/build-with-claude/files) to upload files. You can access the underlying Anthropic client via `provider.client`.

Follow the [OpenAI Files API docs](https://platform.openai.com/docs/api-reference/files/create) to upload files. You can access the underlying OpenAI client via `provider.client`.

Follow the [Google Files API docs](https://ai.google.dev/gemini-api/docs/files) to upload files. You can access the underlying Google GenAI client via `provider.client`.

For Bedrock, files must be uploaded to S3 separately (e.g., using [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3/client/put_object.html)). The assumed role must have `s3:GetObject` permission on the bucket.

Follow the [xAI Files API docs](https://docs.x.ai/docs/guides/files) to upload files. You can access the underlying xAI client via `provider.client`.

# Citations

1. Source page: https://pydantic.dev/docs/ai/core-concepts/input
