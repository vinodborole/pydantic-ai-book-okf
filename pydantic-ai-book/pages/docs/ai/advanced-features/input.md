---
type: Web Page
title: Image, Audio, Video & Document Input | Pydantic Docs
resource: https://pydantic.dev/docs/ai/advanced-features/input
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Image, Audio, Video & Document Input

If you have a direct URL for the image, you can use `ImageUrl`:

If you have the image locally, you can also use `BinaryContent`:

To ensure the example is runnable we download this image from the web, but you can also use `Path().read_bytes()` to read a local file's contents.

You can provide audio input using either `AudioUrl` or `BinaryContent`. The process is analogous to the examples above.

You can provide video input using either `VideoUrl` or `BinaryContent`. The process is analogous to the examples above.

You can provide document input using either `DocumentUrl` or `BinaryContent`. The process is similar to the examples above.

If you have a direct URL for the document, you can use `DocumentUrl`:

The supported document formats vary by model.

You can also use `BinaryContent` to pass document data directly:

You can use `TextContent` to provide text input with additional metadata:

This is equivalent to passing the text as a `str`, but allows you to include additional `metadata` that can be accessed
programmatically in your agent logic.

When using one of `ImageUrl`, `AudioUrl`, `VideoUrl` or `DocumentUrl`, Pydantic AI will default to sending the URL to the model provider, so the file is downloaded on their side.

Support for file URLs varies depending on type and provider:

| Model | Send URL directly | Download and send bytes | Unsupported | 
|---|---|---|---|
| `OpenAIChatModel` | `ImageUrl` | `AudioUrl`,`DocumentUrl` | `VideoUrl`.`DocumentUrl`not supported with`AzureProvider` | 
| `OpenAIResponsesModel` | `ImageUrl`,`AudioUrl`,`DocumentUrl` | — | `VideoUrl` | 
| `AnthropicModel` | `ImageUrl`,`DocumentUrl`(PDF) | `DocumentUrl`(`text/plain`) | `AudioUrl`,`VideoUrl` | 
| `GoogleModel`(Google Cloud) | All URL types | — | — | 
| `GoogleModel`(Gemini API) | YouTube, Files API | All other URLs | — | 
| `XaiModel` | `ImageUrl` | `DocumentUrl` | `AudioUrl`,`VideoUrl` | 
| `MistralModel` | `ImageUrl`,`DocumentUrl`(PDF) | — | `AudioUrl`,`VideoUrl`,`DocumentUrl`(non-PDF) | 
| `BedrockConverseModel` | S3 URLs ( `s3://`) | `ImageUrl`,`DocumentUrl`,`VideoUrl` | `AudioUrl` | 
| `OpenRouterModel` | `ImageUrl`,`DocumentUrl`,`VideoUrl` | `AudioUrl` | — | 

A model API may be unable to download a file (e.g., because of crawling or access restrictions) even if it supports file URLs. For example, `GoogleModel` on Google Cloud limits YouTube video URLs to one URL per request.

In such cases, you can instruct Pydantic AI to download the file content locally and send that instead of the URL by setting `force_download` on the URL object:

Some model providers have their own file storage APIs where you can upload files and reference them by ID or URL.

Use `UploadedFile` to reference files that have been uploaded to a provider’s file storage API.

| Model | Support | 
|---|---|
| `AnthropicModel` | ✅ via Anthropic Files API | 
| `OpenAIChatModel` | ✅ via OpenAI Files API | 
| `OpenAIResponsesModel` | ✅ via OpenAI Files API | 
| `GoogleModel` | ✅ via Google Files API | 
| `BedrockConverseModel` | ✅ via S3 URLs ( `s3://bucket/key`) | 
| `XaiModel` | ✅ via xAI Files API | 
| Other models | ❌ Not supported | 

When using `UploadedFile` you must set the `provider_name`. Uploaded files are specific to the system they are uploaded to and are not transferable across providers. Trying to use a message that contains an `UploadedFile` with a different provider will result in an error.

If you want to introduce portability into your agent logic to allow the same prompt history to work with different provider backends, you can use a history processor to remove or rewrite `UploadedFile` parts from messages before sending them to a provider that does not support them. Be aware that stripping out `UploadedFile` instances might confuse the model, especially if references to those files remain in the text.

The `media_type` parameter is optional for `UploadedFile`. If not specified, Pydantic AI will attempt to infer it from the `file_id`:

- If `file_id`is a URL or path with a recognizable file extension (e.g.,`.pdf`,`.png`), the media type is inferred automatically
- For opaque file IDs (e.g., `'file-abc123'`), the media type defaults to`'application/octet-stream'`

Follow the Anthropic Files API docs to upload files. You can access the underlying Anthropic client via `provider.client`.

Follow the OpenAI Files API docs to upload files. You can access the underlying OpenAI client via `provider.client`.

Follow the Google Files API docs to upload files. You can access the underlying Google GenAI client via `provider.client`.

For Bedrock, files must be uploaded to S3 separately (e.g., using boto3). The assumed role must have `s3:GetObject` permission on the bucket.

Follow the xAI Files API docs to upload files. You can access the underlying xAI client via `provider.client`.

# Citations

1. Source page: https://pydantic.dev/docs/ai/advanced-features/input
