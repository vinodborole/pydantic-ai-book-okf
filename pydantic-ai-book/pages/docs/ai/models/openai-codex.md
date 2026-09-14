---
type: Web Page
title: OpenAI Codex | Pydantic Docs
resource: https://pydantic.dev/docs/ai/models/openai-codex
timestamp: '2026-09-14T12:17:54.595402+00:00'
---

# OpenAI Codex

Use your [ChatGPT/Codex subscription](https://chatgpt.com/codex) with Pydantic AI instead of a pay-per-token API key. The `openai-codex` provider logs in with the same OAuth flow as the official [Codex CLI](https://developers.openai.com/codex/cli/); for API keys, use the [`openai` provider](/docs/ai/models/openai/) instead. Your use of the Codex backend is governed by your agreement with OpenAI; check the applicable [usage policies](https://openai.com/policies/) for your subscription.

To use the Codex provider, you need to either install `pydantic-ai`, or install `pydantic-ai-slim` with the `openai` optional group:

Run `codex login` once with the [Codex CLI](https://developers.openai.com/codex/cli/), then use the `openai-codex:` prefix:

```
from pydantic_ai import Agent
agent = Agent('openai-codex:gpt-5.6-luna')
...
```
This resolves to [`OpenAICodexModel`](/docs/ai/api/models/openai_codex/#pydantic_ai.models.openai_codex.OpenAICodexModel) backed by [`OpenAICodexProvider`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.openai_codex.OpenAICodexProvider), which reads the CLI’s credentials from `~/.codex/auth.json` (or `$CODEX_HOME/auth.json`). The file is never written to; refreshed tokens live in memory for the rest of the process.

If you don’t want to depend on the Codex CLI, [`OpenAICodexOAuthFlow`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.openai_codex.OpenAICodexOAuthFlow) runs the same browser login. The Codex client pins its redirect URI to `http://localhost:1455/auth/callback`, so [`exchange_code_from_callback()`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.openai_codex.OpenAICodexOAuthFlow.exchange_code_from_callback) listens on that port until the browser redirects there, then exchanges the code for credentials:

Passing `credentials` keeps them in memory only, so the next process has to log in again. To log in once, persist them as described below.

The provider refreshes expired tokens automatically, and refresh tokens are single-use, so the stored copy has to keep up. Give the provider an [`OpenAICodexCredentialSource`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.openai_codex.OpenAICodexCredentialSource) and it calls `load()` on first use and `save()` after every refresh. Run the login flow only when the store is empty:

If `save()` raises, the refreshed credentials stay live in memory and a [`CredentialsPersistenceError`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.openai_codex.CredentialsPersistenceError) is raised. Both it and [`CredentialsRefreshError`](/docs/ai/api/pydantic-ai/providers/#pydantic_ai.providers.openai_codex.CredentialsRefreshError) subclass [`ModelAPIError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.ModelAPIError), so a [`FallbackModel`](/docs/ai/api/models/fallback/#pydantic_ai.models.fallback.FallbackModel) treats an unusable login like any other provider failure.

[Logfire instrumentation](/docs/ai/integrations/logfire/) can trace agent runs without capturing OAuth credentials. Leave HTTP body capture disabled unless you need it: `logfire.instrument_httpx(capture_all=True)` captures authorization codes and token responses, which require additional scrubbing patterns.

If you enable full HTTP capture, configure scrubbing before starting the OAuth flow:

The additional patterns redact the OAuth credentials and account identifiers; Logfire’s default patterns already redact the authorization header.

To mirror the official Codex client’s prompt-cache affinity, [`OpenAICodexModel`](/docs/ai/api/models/openai_codex/#pydantic_ai.models.openai_codex.OpenAICodexModel) sends the `session-id`, `thread-id`, and `x-client-request-id` headers and the `prompt_cache_key` request field. All four are derived from the [`conversation_id`](/docs/ai/core-concepts/message-history/) of the message history, so runs continuing the same conversation reuse a stable identity. An explicit `openai_prompt_cache_key` model setting or explicitly supplied `extra_headers` always win over the derived values. This does not guarantee a cache hit.

- The Codex backend is streaming-only; for non-streaming runs the library transparently drains a stream, so `agent.run_sync()` and friends work as usual.
- Unsupported generic settings (`max_tokens` ,`temperature` , and`top_p` ) are dropped before sending. The Codex profile leaves explicit`openai_top_logprobs` ,`openai_truncation` , and`openai_user` settings to the standard OpenAI handling, so backend incompatibilities surface as errors. The usual reasoning-related restrictions on log probabilities still apply.
- The backend requires `store=false` , so every request is sent with it and an explicit`openai_store=True` is silently overridden: responses are never persisted server-side. Consequently, resuming a suspended run raises[`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) , since there is no stored response to continue from.
- `count_tokens()` raises[`UserError`](/docs/ai/api/pydantic-ai/exceptions/#pydantic_ai.exceptions.UserError) : the input-tokens endpoint is not served under subscription auth.
- There is no device flow: the browser login above is the only login flow the Codex client supports.

# Citations

1. Source page: https://pydantic.dev/docs/ai/models/openai-codex
