---
type: Web Page
title: Thinking | Pydantic Docs
resource: https://pydantic.dev/docs/ai/capabilities/thinking
timestamp: '2026-08-03T09:54:19.663642+00:00'
---

# Thinking

Thinking (or reasoning) is the process by which a model works through a problem step-by-step before providing its final answer.

The simplest way to enable thinking across supported providers is the `Thinking`[capability](/docs/ai/capabilities/overview).
Provider-specific settings are available for advanced usage when you need direct access to a provider’s native thinking controls.

Use the [`Thinking`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Thinking) capability to enable thinking:

You can also set the underlying `thinking` field in [`ModelSettings`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings) directly:

The [`Thinking.effort`](/docs/ai/api/pydantic-ai/capabilities/#pydantic_ai.capabilities.Thinking.effort) value accepts:

- `True` — enable thinking with the provider’s default effort level
- `False` — disable thinking (silently ignored on always-on models)
- `'minimal'` /`'low'` /`'medium'` /`'high'` /`'xhigh'` — enable thinking at a specific effort level (unsupported levels map to the closest available value)

These are the same values accepted by the underlying `thinking` model setting.
When omitted, the model uses its default behavior. Provider-specific settings (documented in the sections below) take precedence when both are set.

The `Thinking` capability maps each effort value to the selected provider’s native format:

| Provider | `Thinking()` /`Thinking(effort=True)` | `Thinking(effort='high')` | Notes | 
|---|---|---|---|
| Anthropic (Opus 4.6+) | `anthropic_thinking={'type': 'adaptive'}` | `{type: 'adaptive'}` +`effort='high'` | Claude Opus 4.7, 4.8, 5, and Sonnet 5 also support `effort='xhigh'` | 
| Anthropic (older) | `anthropic_thinking={'type': 'enabled', 'budget_tokens': 10000}` | `budget_tokens=16384` | Budget-based; `'low'` → 2048 tokens | 
| OpenAI | `reasoning_effort='medium'` | `reasoning_effort='high'` |  | 
| Google (Gemini 3+) | `include_thoughts=True` | `thinking_level='HIGH'` |  | 
| Google (Gemini 2.5) | `include_thoughts=True` | `thinking_budget=24576` |  | 
| Groq | `reasoning_format='parsed'` (gpt-oss also`reasoning_effort='medium'` ) | `reasoning_format='parsed'` (gpt-oss also`reasoning_effort='high'` ) | gpt-oss: unified effort → `reasoning_effort` (`low` /`medium` /`high` , via`extra_body` ; always-on, so`thinking=False` is silently ignored); qwen3:`thinking=False` →`reasoning_effort='none'` (true disable, via`extra_body` ); other reasoning models →`'hidden'` (suppresses output only) | 
| Mistral | `reasoning_effort='high'` | `reasoning_effort='high'` | Only on adjustable-reasoning models (e.g. `mistral-small-latest` ,`mistral-medium-3-5` );`magistral` reasons always-on and gets no`reasoning_effort` . Mistral exposes only`'high'` /`'none'` , so every enabled level (incl.`'minimal'` ) →`'high'` and only`thinking=False` →`'none'` | 
| OpenRouter | `reasoning={'effort': 'medium', 'enabled': True}` | `reasoning={'effort': 'high', 'enabled': True}` | `thinking=False` →`effort='none'` ; always-on routes silently ignore; via`extra_body` | 
| Cerebras | `reasoning_effort` omitted (reasons by default) | `reasoning_effort` omitted | `thinking=False` →`reasoning_effort='none'` ; gpt-oss reasons always-on, so`thinking=False` is silently ignored | 
| xAI | `reasoning_effort` omitted on Grok 4.3 (uses its default) | `reasoning_effort='high'` | Grok 4.3 supports `'none'` ,`'low'` ,`'medium'` , and`'high'` , and`thinking=True` omits the parameter so the model applies its own default; Grok 3 Mini only supports`'low'` and`'high'` (so`thinking=True` →`'high'` ) and silently ignores`thinking=False` ; Grok 4.5 supports`'low'` ,`'medium'` , and`'high'` but not`'none'` , so it reasons always-on (`thinking=True` →`'medium'` ) and silently ignores`thinking=False` | 
| Bedrock (Claude 4.6+) | `thinking.type='adaptive'` | `{type: 'adaptive'}` +`output_config.effort='high'` | Effort lives in the sibling `output_config` field per AWS docs;`xhigh` maps to`max` | 
| Bedrock (Claude older) | `thinking.type='enabled'` | `budget_tokens=16384` | Budget-based | 
| Bedrock (OpenAI) | `reasoning_effort='medium'` | `reasoning_effort='high'` | Converse rejects `'none'` ;`thinking=False` silently ignored | 
| Bedrock (Qwen) | `reasoning_config='high'` | `reasoning_config='high'` | Only `'low'` and`'high'` ;`thinking=False` silently ignored | 

When using the [`OpenAIChatModel`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIChatModel), text output inside `<think>` tags are converted to [`ThinkingPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ThinkingPart) objects.
You can customize the tags using the [`thinking_tags`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.ModelProfile.thinking_tags) field on the [model profile](/docs/ai/models/openai#model-profile).

Some [OpenAI-compatible model providers](/docs/ai/models/openai#openai-compatible-models) might also support native thinking parts that are not delimited by tags. Instead, they are sent and received as separate, custom fields in the API. Typically, if you are calling the model via the `<provider>:<model>` shorthand, Pydantic AI handles it for you. Nonetheless, you can still configure the fields with [`openai_chat_thinking_field`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.openai.OpenAIModelProfile.openai_chat_thinking_field).

If your provider recommends to send back these custom fields not changed, for caching or interleaved thinking benefits, you can also achieve this with [`openai_chat_send_back_thinking_parts`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.openai.OpenAIModelProfile.openai_chat_send_back_thinking_parts).

The [`OpenAIResponsesModel`](/docs/ai/api/models/openai/#pydantic_ai.models.openai.OpenAIResponsesModel) can generate native thinking parts.
To enable this functionality, you need to set the
`OpenAIResponsesModelSettings.openai_reasoning_effort` and `OpenAIResponsesModelSettings.openai_reasoning_summary`[model settings](/docs/ai/core-concepts/agent#model-run-settings).
Models that support it can additionally use a `pro` [reasoning mode](/docs/ai/models/openai#reasoning-mode), which is independent of the effort and never set by the unified `thinking` setting.

By default, the unique IDs of reasoning, text, and function call parts from the message history are sent to the model, which can result in errors like `"Item 'rs_123' of type 'reasoning' was provided without its required following item."`
if the message history you’re sending does not match exactly what was received from the Responses API in a previous response, for example if you’re using a [history processor](/docs/ai/core-concepts/message-history#processing-message-history).
To disable this, you can disable the `OpenAIResponsesModelSettings.openai_send_reasoning_ids`[model setting](/docs/ai/core-concepts/agent#model-run-settings).

To enable thinking, use the `AnthropicModelSettings.anthropic_thinking`[model setting](/docs/ai/core-concepts/agent#model-run-settings).

Anthropic reports how many thinking tokens it used in [`RunUsage.details`](/docs/ai/api/pydantic-ai/usage/#pydantic_ai.usage.RunUsage.details) under the `thinking_tokens` key. They are billed within `output_tokens`, so they are a readable subset of the output total rather than an addition to it, and the key is omitted entirely when a response used no thinking tokens.

To enable [interleaved thinking](https://docs.anthropic.com/en/docs/build-with-claude/extended-thinking#interleaved-thinking), you need to include the beta header in your model settings:

Starting with `claude-opus-4-6`, Anthropic supports [adaptive thinking](https://docs.anthropic.com/en/docs/build-with-claude/adaptive-thinking), where the model dynamically decides when and how much to think based on the complexity of each request. This replaces extended thinking (`type: 'enabled'` with `budget_tokens`) which is deprecated on Opus 4.6 and removed on Opus 4.7, 4.8, 5, and Sonnet 5. Claude Opus 4.7, 4.8, 5, and Sonnet 5 also add the `xhigh` effort level. Adaptive thinking also automatically enables interleaved thinking.

The [`anthropic_effort`](/docs/ai/api/models/anthropic/#pydantic_ai.models.anthropic.AnthropicModelSettings.anthropic_effort) setting controls how much effort the model puts into its response (independent of thinking). See the [Anthropic effort docs](https://docs.anthropic.com/en/docs/build-with-claude/effort) for details.

Thinking tokens count against Anthropic’s loop-wide [task budgets](/docs/ai/models/anthropic#task-budgets-beta), so adaptive thinking naturally scales down as the budget depletes.

For advanced usage, use the `GoogleModelSettings.google_thinking_config`[model setting](/docs/ai/core-concepts/agent#model-run-settings).

See the [Google model docs](/docs/ai/models/google#configure-thinking) for more details.

xAI reasoning models (Grok) support native thinking. To preserve the thinking content for multi-turn conversations, enable [`XaiModelSettings.xai_include_encrypted_content`](/docs/ai/api/models/xai/#pydantic_ai.models.xai.XaiModelSettings.xai_include_encrypted_content).

For Claude Sonnet 4.6+ and Opus 4.6+, Pydantic AI’s unified `thinking` setting translates to AWS’s required [adaptive thinking](https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-adaptive-thinking.html) shape automatically — set [`ModelSettings.thinking`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings.thinking) and you’re done.

For older Claude models or to pin a specific `budget_tokens`, you can still use `BedrockModelSettings.bedrock_additional_model_requests_fields`[model setting](/docs/ai/core-concepts/agent#model-run-settings) to pass provider-specific configuration directly:

Reasoning is [always enabled](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-reasoning.html) for Deepseek model

Groq supports different formats to receive thinking parts:

- `"raw"` : The thinking part is included in the text content inside`<think>` tags, which are automatically converted to[`ThinkingPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ThinkingPart) objects.
- `"hidden"` : The thinking part is not included in the text content.
- `"parsed"` : The thinking part has its own structured part in the response which is converted into a[`ThinkingPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ThinkingPart) object.

The unified [`ModelSettings.thinking`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings.thinking) setting works across providers: it selects `reasoning_format='parsed'` so thinking parts are returned, and for the gpt-oss family its effort level also drives Groq’s `reasoning_effort` (`minimal`/`low` → `'low'`, `medium` → `'medium'`, `high`/`xhigh` → `'high'`, `True` → `'medium'`).

Two composable [model settings](/docs/ai/core-concepts/agent#model-run-settings) give finer control: [`GroqModelSettings.groq_reasoning_format`](/docs/ai/api/models/groq/#pydantic_ai.models.groq.GroqModelSettings.groq_reasoning_format) selects how thinking parts are returned (the formats above), and [`GroqModelSettings.groq_reasoning_effort`](/docs/ai/api/models/groq/#pydantic_ai.models.groq.GroqModelSettings.groq_reasoning_effort) (sent to Groq as `reasoning_effort`) controls how much the model reasons, taking precedence over the unified `thinking` mapping:

To enable thinking, use the `OpenRouterModelSettings.openrouter_reasoning`[model setting](/docs/ai/core-concepts/agent#model-run-settings).

To enable thinking, use the unified `thinking`[model setting](/docs/ai/core-concepts/agent#model-run-settings). To preserve thinking content across multi-turn conversations, also set `ZaiModelSettings.zai_clear_thinking` to `False`.

The `magistral` family always reasons and does not need to be specifically enabled; `thinking=False` is silently ignored. Mistral has [deprecated](https://docs.mistral.ai/resources/deprecated/native-reasoning) the `magistral` family in favor of the adjustable-reasoning models below.

Models with adjustable reasoning (the Mistral Small 4 and Medium 3.5 families: `mistral-small-latest`, `mistral-small-2603`, `mistral-medium-latest`, `mistral-medium`, `mistral-medium-3`, `mistral-medium-3-5`, `mistral-medium-3.5`, `mistral-medium-2604`) are controlled via the unified [`thinking`](/docs/ai/api/pydantic-ai/settings/#pydantic_ai.settings.ModelSettings.thinking) setting, which maps to Mistral’s `reasoning_effort`. Mistral exposes only `'high'` (full thinking) and `'none'` (thinking suppressed), so every enabled level maps to `'high'` and only `thinking=False` maps to `'none'`. Older `mistral-small-*` / `mistral-medium-*` snapshots do not support reasoning, so `thinking` is silently ignored for them. Adjustable reasoning applies when using the native Mistral provider; OpenAI-compatible providers that host these models (such as LiteLLM or Azure) do not support it and `thinking` is ignored there. OpenRouter is the exception: it maps the unified `thinking` setting to its own `reasoning` parameter for any model it routes.

Thinking is supported by the `command-a-reasoning-08-2025` model. It does not need to be specifically enabled.

Text output inside `<think>` tags is automatically converted to [`ThinkingPart`](/docs/ai/api/pydantic-ai/messages/#pydantic_ai.messages.ThinkingPart) objects.
You can customize the tags using the [`thinking_tags`](/docs/ai/api/pydantic-ai/profiles/#pydantic_ai.profiles.ModelProfile.thinking_tags) field on the [model profile](/docs/ai/models/openai#model-profile).

# Citations

1. Source page: https://pydantic.dev/docs/ai/capabilities/thinking
