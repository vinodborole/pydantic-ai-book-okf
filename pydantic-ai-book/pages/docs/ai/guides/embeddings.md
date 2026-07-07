---
type: Web Page
title: Embeddings | Pydantic Docs
resource: https://pydantic.dev/docs/ai/guides/embeddings
timestamp: '2026-07-07T10:31:51.511921+00:00'
---

# Embeddings

Embeddings are vector representations of text that capture semantic meaning. They’re essential for building:

- **Semantic search**— Find documents based on meaning, not just keyword matching
- **RAG (Retrieval-Augmented Generation)**— Retrieve relevant context for your AI agents
- **Similarity detection**— Find similar documents, detect duplicates, or cluster content
- **Classification**— Use embeddings as features for downstream ML models

Pydantic AI provides a unified interface for generating embeddings across multiple providers.

The `Embedder` class is the high-level interface for generating embeddings:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

All embed methods return an `EmbeddingResult` containing the embeddings along with useful metadata.

For convenience, you can access embeddings either by index (`result[0]`) or by the original input text (`result['Hello world']`).

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

The best embedding model depends on your constraints. Here’s a starting-point cheat sheet; consult each provider’s docs and the MTEB leaderboard before committing to a model for a large index.

| If you want… | For example | 
|---|---|
| A managed API | `openai:text-embedding-3-small`(cheap default),`openai:text-embedding-3-large`,`voyageai:voyage-3.5`, or`cohere:embed-v4.0` | 
| No API key, private, free | `sentence-transformers:google/embeddinggemma-300m`,`sentence-transformers:lightonai/DenseOn`,`sentence-transformers:Qwen/Qwen3-Embedding-0.6B`, or any other Hugging Face model | 
| Multilingual | `cohere:embed-multilingual-v3.0`,`sentence-transformers:jinaai/jina-embeddings-v5-text-small-retrieval`, or`sentence-transformers:Snowflake/snowflake-arctic-embed-l-v2.0` | 
| Specialized domain | `voyageai:voyage-code-3`,`voyageai:voyage-law-2`,`voyageai:voyage-finance-2`,`sentence-transformers:nomic-ai/CodeRankEmbed`, or`sentence-transformers:TechWolf/JobBERT-v3` | 
| To run on AWS infra you already have | `bedrock:amazon.titan-embed-text-v2:0`or`bedrock:cohere.embed-v4:0` | 
| To reduce index size | Any model with dimension control (see Settings) | 

`OpenAIEmbeddingModel` works with OpenAI’s embeddings API and any OpenAI-compatible provider.

To use OpenAI embedding models, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `openai` optional group:

To use `OpenAIEmbeddingModel` with the OpenAI API, go to platform.openai.com and follow your nose until you find the place to generate an API key. Once you have the API key, you can set it as an environment variable:

You can then use the model:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

See OpenAI’s embedding models for available models.

OpenAI’s `text-embedding-3-*` models support dimension reduction via the `dimensions` setting:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Since `OpenAIEmbeddingModel` uses the same provider system as `OpenAIChatModel`, you can use it with any OpenAI-compatible provider:

For providers with dedicated provider classes (like `OllamaProvider` or `AzureProvider`), you can use the shorthand syntax:

```
from pydantic_ai import Embedder
embedder = Embedder('azure:text-embedding-3-small')
embedder = Embedder('ollama:nomic-embed-text')
```
See OpenAI-compatible Models for the full list of supported providers.

`GoogleEmbeddingModel` works with Google’s embedding models via the Gemini API (Google AI Studio) or Google Cloud (formerly known as Vertex AI).

To use Google embedding models, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `google` optional group:

To use `GoogleEmbeddingModel` with the Gemini API, go to aistudio.google.com and generate an API key. Once you have the API key, you can set it as an environment variable:

You can then use the model:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

See the Google Embeddings documentation for available models.

To use Google’s embedding models via Google Cloud (formerly known as Vertex AI) instead of the Gemini API, use the `google-cloud:` provider prefix:

See the Google provider documentation for more details on Google Cloud authentication options, including application default credentials, service accounts, and API keys.

Google’s embedding models support dimension reduction via the `dimensions` setting:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

`gemini-embedding-2` is conditioned on the task you’re embedding for by prepending a short task instruction to the input text, rather than through the `google_task_type` field used by the other Google models. Pydantic AI builds this prefix for you via the `google_task` setting:

`google_task` accepts the task names from Google’s API: `'search result'`, `'question answering'`, `'fact checking'`, `'code retrieval'`, `'classification'`, `'clustering'`, and `'sentence similarity'`. For the retrieval-style (asymmetric) tasks, queries and documents are prefixed differently, so the same task applies to both sides of a pair; the remaining (symmetric) tasks prefix both inputs the same way.

When you don’t set `google_task`, `gemini-embedding-2` is conditioned as `'search result'`. Conditioning is on by default because Google recommends it for this model and it yields better retrieval performance than embedding raw text. To opt out and embed the text verbatim, pass `google_task='raw'`.

`google_task` only applies to `gemini-embedding-2`; on any other model it is ignored with a warning (those models condition via `google_task_type` instead). Conversely, `google_task_type` is ignored on `gemini-embedding-2`, since that model conditions through the text prefix.

Google models support additional settings via `GoogleEmbeddingSettings`:

See Google’s task type documentation for available task types. By default, `embed_query()` uses `RETRIEVAL_QUERY` and `embed_documents()` uses `RETRIEVAL_DOCUMENT`.

`CohereEmbeddingModel` provides access to Cohere’s embedding models, which offer multilingual support and various model sizes.

To use Cohere embedding models, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `cohere` optional group:

To use `CohereEmbeddingModel`, go to dashboard.cohere.com/api-keys and follow your nose until you find the place to generate an API key. Once you have the API key, you can set it as an environment variable:

You can then use the model:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

See the Cohere Embed documentation for available models.

Cohere models support additional settings via `CohereEmbeddingSettings`:

`VoyageAIEmbeddingModel` provides access to VoyageAI’s embedding models, which are optimized for retrieval with specialized models for code, finance, and legal domains.

To use VoyageAI embedding models, you need to install `pydantic-ai-slim` with the `voyageai` optional group:

To use `VoyageAIEmbeddingModel`, go to dash.voyageai.com to generate an API key. Once you have the API key, you can set it as an environment variable:

You can then use the model:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

See the VoyageAI Embeddings documentation for available models.

VoyageAI models support additional settings via `VoyageAIEmbeddingSettings`:

`BedrockEmbeddingModel` provides access to embedding models through AWS Bedrock, including Amazon Titan, Cohere, and Amazon Nova models.

To use Bedrock embedding models, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `bedrock` optional group:

Authentication with AWS Bedrock uses standard AWS credentials. See the Bedrock provider documentation for details on configuring credentials via environment variables, AWS credentials file, or IAM roles.

Ensure your AWS account has access to the Bedrock embedding models you want to use. See AWS Bedrock model access for details.

*(This example requires AWS credentials configured)*

Bedrock supports three families of embedding models. See the AWS Bedrock documentation for the full list of available models.

**Amazon Titan:**

- `amazon.titan-embed-text-v1`— 1536 dimensions (fixed), 8K tokens
- `amazon.titan-embed-text-v2:0`— 256/384/1024 dimensions (configurable, default: 1024), 8K tokens

**Cohere Embed:**

- `cohere.embed-english-v3`— English-only, 1024 dimensions (fixed), 512 tokens
- `cohere.embed-multilingual-v3`— Multilingual, 1024 dimensions (fixed), 512 tokens
- `cohere.embed-v4:0`— 256/512/1024/1536 dimensions (configurable, default: 1536), 128K tokens

**Amazon Nova:**

- `amazon.nova-2-multimodal-embeddings-v1:0`— 256/384/1024/3072 dimensions (configurable, default: 3072), 8K tokens

Titan v2 supports vector normalization for direct similarity calculations via `bedrock_titan_normalize` (default: `True`). Titan v1 does not support this setting.

Cohere models on Bedrock support additional settings via `BedrockEmbeddingSettings`:

- `bedrock_cohere_input_type`— By default,- `embed_query()`uses- `'search_query'`and- `embed_documents()`uses- `'search_document'`. Also accepts- `'classification'`or- `'clustering'`.
- `bedrock_cohere_truncate`— Fine-grained truncation control:- `'NONE'`(default, error on overflow),- `'START'`, or- `'END'`. Overrides the base- `truncate`setting.
- `bedrock_cohere_max_tokens`— Limits tokens per input (default: 128000). Only supported by Cohere v4.

Nova models on Bedrock support additional settings via `BedrockEmbeddingSettings`:

- `bedrock_nova_truncate`— Fine-grained truncation control:- `'NONE'`(default, error on overflow),- `'START'`, or- `'END'`. Overrides the base- `truncate`setting.
- `bedrock_nova_embedding_purpose`— By default,- `embed_query()`uses- `'GENERIC_RETRIEVAL'`and- `embed_documents()`uses- `'GENERIC_INDEX'`. Also accepts- `'TEXT_RETRIEVAL'`,- `'CLASSIFICATION'`, or- `'CLUSTERING'`.

Models that don’t support batch embedding (Titan and Nova) make individual API requests for each input text. By default, these requests run concurrently with a maximum of 5 parallel requests.

You can adjust this with the `bedrock_max_concurrency` setting:

Bedrock supports cross-region inference using geographic prefixes like `us.`, `eu.`, or `apac.`:

Set `bedrock_inference_profile` to route requests through an inference profile while keeping the base model name for detecting model capabilities:

For advanced configuration like explicit credentials or a custom boto3 client, you can create a `BedrockProvider` directly. See the Bedrock provider documentation for more details.

`SentenceTransformerEmbeddingModel` runs embeddings locally using the sentence-transformers library, giving you access to the thousands of embedding models on Hugging Face without any API calls. This is ideal for:

- **Privacy**— Data never leaves your infrastructure
- **Cost**— No API charges for high-volume workloads
- **Offline use**— No internet connection required after model download
- **Specialized domains or languages**- Pick models trained for code, multilingual, biomedical, legal, etc. from the MTEB leaderboard

To use Sentence Transformers embedding models, you need to install `pydantic-ai-slim` with the `sentence-transformers` optional group:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

`lightonai/DenseOn` is a strong recent 149M-parameter general-purpose model that encodes queries and documents asymmetrically: `embed_query()` and `embed_documents()` automatically apply the model’s `query:` / `document:` prompts. See the Sentence Transformers pretrained models documentation and the MTEB leaderboard for more options; see also Choosing a model above.

Control which device to use for inference:

If you need more control over model initialization:

`EmbeddingSettings` provides common configuration options that work across providers:

- `dimensions`: Reduce the output embedding dimensions (supported by OpenAI, Google, Cohere, Bedrock, VoyageAI)
- `truncate`: When- `True`, truncate input text that exceeds the model’s context length instead of raising an error (supported by Cohere, Bedrock, VoyageAI)

Settings can be specified at the embedder level (applied to all calls) or per-call:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

You can check token counts before embedding to avoid exceeding model limits:

*(This example is complete, it can be run “as is” — you’ll need to add  asyncio.run(main()) to run main)*

Use `TestEmbeddingModel` for testing without making API calls:

Enable OpenTelemetry instrumentation for debugging and monitoring:

See the Debugging and Monitoring guide for more details on using Logfire with Pydantic AI.

For high-quality retrieval, a common pattern is **two-stage**: first use an embedding model to pull a broad shortlist of candidates cheaply, then use a **cross-encoder reranker** to score each candidate against the query more precisely. The cross-encoder reads the query and document *together*, so it’s slower than an embedding lookup but dramatically more accurate, making it ideal for narrowing a top-100 recall list down to the top-5 results you actually hand to the LLM.

Pydantic AI does not ship a reranker provider class, so you bring your own. The most common local option is a `CrossEncoder` from `sentence-transformers`:

Call `rerank()` on the candidates returned by your vector search (for example, in the `retrieve` tool of the RAG example) before handing the results to the LLM.

To integrate a custom embedding provider, subclass `EmbeddingModel`:

Use `WrapperEmbeddingModel` if you want to wrap an existing model to add custom behavior like caching or logging.

# Citations

1. Source page: https://pydantic.dev/docs/ai/guides/embeddings
