---
title: Provider Routing
subtitle: Route requests to the best provider
headline: Provider Routing | Intelligent Multi-Provider Request Routing
canonical-url: 'https://openrouter.ai/docs/features/provider-routing'
'og:site_name': OpenRouter Documentation
'og:title': Provider Routing - Smart Multi-Provider Request Management
'og:description': >-
  Route AI model requests across multiple providers intelligently. Learn how to
  optimize for cost, performance, and reliability with OpenRouter's provider
  routing.
'og:image':
  type: url
  value: >-
    https://openrouter.ai/dynamic-og?pathname=features/provider-routing&title=Smart%20Routing&description=Optimize%20AI%20requests%20across%20providers%20for%20best%20performance
'og:image:width': 1200
'og:image:height': 630
'twitter:card': summary_large_image
'twitter:site': '@OpenRouterAI'
noindex: false
nofollow: false
---

import { ProviderPreferencesSchema } from '../../../imports/constants';
import { TSFetchCodeBlock } from '../../../imports/TSFetchCodeBlock';
import { ZodToJSONSchemaBlock } from '../../../imports/ZodToJSONSchemaBlock';

OpenRouter routes requests to the best available providers for your model. By default, [requests are load balanced](#load-balancing-default-strategy) across the top providers to maximize uptime.

You can customize how your requests are routed using the `provider` object in the request body for [Chat Completions](/docs/api-reference/chat-completion) and [Completions](/docs/api-reference/completion).

<Tip>
  For a complete list of valid provider names to use in the API, see the [full
  provider schema](#json-schema-for-provider-preferences).
</Tip>

The `provider` object can contain the following fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `order` | string[] | - | List of provider names to try in order (e.g. `["Anthropic", "OpenAI"]`). [Learn more](#ordering-specific-providers) |
| `allow_fallbacks` | boolean | `true` | Whether to allow backup providers when the primary is unavailable. [Learn more](#disabling-fallbacks) |
| `require_parameters` | boolean | `false` | Only use providers that support all parameters in your request. [Learn more](#requiring-providers-to-support-all-parameters-beta) |
| `data_collection` | "allow" \| "deny" | "allow" | Control whether to use providers that may store data. [Learn more](#requiring-providers-to-comply-with-data-policies) |
| `ignore` | string[] | - | List of provider names to skip for this request. [Learn more](#ignoring-providers) |
| `quantizations` | string[] | - | List of quantization levels to filter by (e.g. `["int4", "int8"]`). [Learn more](#quantization) |
| `sort` | string | - | Sort providers by price or throughput. (e.g. `"price"` or `"throughput"`). [Learn more](#provider-sorting) |

## Price-Based Load Balancing (Default Strategy)

For each model in your request, OpenRouter's default behavior is to load balance requests across providers, prioritizing price.

If you are more sensitive to throughput than price, you can use the `sort` field to explicitly prioritize throughput.

<Tip>
  When you send a request with `tools` or `tool_choice`, OpenRouter will only
  route to providers that support tool use. Similarly, if you set a
  `max_tokens`, then OpenRouter will only route to providers that support a
  response of that length.
</Tip>

Here is OpenRouter's default load balancing strategy:

1. Prioritize providers that have not seen significant outages in the last 30 seconds.
2. For the stable providers, look at the lowest-cost candidates and select one weighted by inverse square of the price (example below).
3. Use the remaining providers as fallbacks.

<Note title="A Load Balancing Example">
If Provider A costs \$1 per million tokens, Provider B costs \$2, and Provider C costs \$3, and Provider B recently saw a few outages.

- Your request is routed to Provider A. Provider A is 9x more likely to be first routed to Provider A than Provider C because $(1 / 3^2 = 1/9)$ (inverse square of the price).
- If Provider A fails, then Provider C will be tried next.
- If Provider C also fails, Provider B will be tried last.

</Note>

If you have `sort` or `order` set in your provider preferences, load balancing will be disabled.

## Provider Sorting

As described above, OpenRouter load balances based on price, while taking uptime into account.

If you instead want to _explicitly_ prioritize a particular provider attribute, you can include the `sort` field in the `provider` preferences. Load balancing will be disabled, and the router will try providers in order.

The three sort options are:

- `"price"`: prioritize lowest price
- `"throughput"`: prioritize highest throughput
- `"latency"`: prioritize lowest latency

<TSFetchCodeBlock
  title='Example with Fallbacks Enabled'
  uriPath='/api/v1/chat/completions'
  body={{
    model: 'meta-llama/llama-3.1-70b-instruct',
    messages: [{ role: 'user', content: 'Hello' }],
    provider: {
      sort: 'throughput',
    },
  }}
/>

To _always_ prioritize low prices, and not apply any load balancing, set `sort` to `"price"`.

To _always_ prioritize low latency, and not apply any load balancing, set `sort` to `"latency"`.

## Nitro Shortcut

You can append `:nitro` to any model slug as a shortcut to sort by throughput. This is exactly equivalent to setting `provider.sort` to `"throughput"`.

<TSFetchCodeBlock
  title='Example using Nitro shortcut'
  uriPath='/api/v1/chat/completions'
  body={{
    model: 'meta-llama/llama-3.1-70b-instruct:nitro',
    messages: [{ role: 'user', content: 'Hello' }],
  }}
/>

## Floor Price Shortcut

You can append `:floor` to any model slug as a shortcut to sort by price. This is exactly equivalent to setting `provider.sort` to `"price"`.

<TSFetchCodeBlock
  title='Example using Floor shortcut'
  uriPath='/api/v1/chat/completions'
  body={{
    model: 'meta-llama/llama-3.1-70b-instruct:floor',
    messages: [{ role: 'user', content: 'Hello' }],
  }}
/>

## Ordering Specific Providers

You can set the providers that OpenRouter will prioritize for your request using the `order` field.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `order` | string[] | - | List of provider names to try in order (e.g. `["Anthropic", "OpenAI"]`). |

The router will prioritize providers in this list, and in this order, for the model you're using. If you don't set this field, the router will [load balance](#load-balancing-default-strategy) across the top providers to maximize uptime.

OpenRouter will try them one at a time and proceed to other providers if none are operational. If you don't want to allow any other providers, you should [disable fallbacks](#disabling-fallbacks) as well.

### Example: Specifying providers with fallbacks

This example skips over OpenAI (which doesn't host Mixtral), tries Together, and then falls back to the normal list of providers on OpenRouter:

<TSFetchCodeBlock
  title='Example with Fallbacks Enabled'
  uriPath='/api/v1/chat/completions'
  body={{
    model: 'mistralai/mixtral-8x7b-instruct',
    messages: [{ role: 'user', content: 'Hello' }],
    provider: {
      order: ['OpenAI', 'Together'],
    },
  }}
/>

### Example: Specifying providers with fallbacks disabled

Here's an example with `allow_fallbacks` set to `false` that skips over OpenAI (which doesn't host Mixtral), tries Together, and then fails if Together fails:

<TSFetchCodeBlock
  title='Example with Fallbacks Disabled'
  uriPath='/api/v1/chat/completions'
  body={{
    model: 'mistralai/mixtral-8x7b-instruct',
    messages: [{ role: 'user', content: 'Hello' }],
    provider: {
      order: ['OpenAI', 'Together'],
      allow_fallbacks: false,
    },
  }}
/>

## Requiring Providers to Support All Parameters (beta)

You can restrict requests only to providers that support all parameters in your request using the `require_parameters` field.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `require_parameters` | boolean | `false` | Only use providers that support all parameters in your request. |

With the default routing strategy, providers that don't support all the [LLM parameters](/docs/api-reference/parameters) specified in your request can still receive the request, but will ignore unknown parameters. When you set `require_parameters` to `true`, the request won't even be routed to that provider.

### Example: Excluding providers that don't support JSON formatting

For example, to only use providers that support JSON formatting:

<TSFetchCodeBlock
  uriPath='/api/v1/chat/completions'
  body={{
    messages: [{ role: 'user', content: 'Hello' }],
    provider: {
      require_parameters: true,
    },
    response_format: { type: 'json_object' },
  }}
/>

## Requiring Providers to Comply with Data Policies

You can restrict requests only to providers that comply with your data policies using the `data_collection` field.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `data_collection` | "allow" \| "deny" | "allow" | Control whether to use providers that may store data. |

- `allow`: (default) allow providers which store user data non-transiently and may train on it
- `deny`: use only providers which do not collect user data

Some model providers may log prompts, so we display them with a **Data Policy** tag on model pages. This is not a definitive source of third party data policies, but represents our best knowledge.

<Tip title='Account-Wide Data Policy Filtering'>
  This is also available as an account-wide setting in [your privacy
  settings](https://openrouter.ai/settings/privacy). You can disable third party
  model providers that store inputs for training.
</Tip>

### Example: Excluding providers that don't comply with data policies

To exclude providers that don't comply with your data policies, set `data_collection` to `deny`:

<TSFetchCodeBlock
  uriPath='/api/v1/chat/completions'
  body={{
    messages: [{ role: 'user', content: 'Hello' }],
    provider: {
      data_collection: 'deny', // or "allow"
    },
  }}
/>

## Disabling Fallbacks

To guarantee that your request is only served by the top (lowest-cost) provider, you can disable fallbacks.

This is combined with the `order` field from [Ordering Specific Providers](#ordering-specific-providers) to restrict the providers that OpenRouter will prioritize to just your chosen list.

<TSFetchCodeBlock
  uriPath='/api/v1/chat/completions'
  body={{
    messages: [{ role: 'user', content: 'Hello' }],
    provider: {
      allow_fallbacks: false,
    },
  }}
/>

## Ignoring Providers

You can ignore providers for a request by setting the `ignore` field in the `provider` object.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `ignore` | string[] | - | List of provider names to skip for this request. |

<Warning>
  Ignoring multiple providers may significantly reduce fallback options and
  limit request recovery.
</Warning>

<Tip title="Account-Wide Ignored Providers">
You can ignore providers for all account requests by configuring your [preferences](/settings/preferences). This configuration applies to all API requests and chatroom messages.

Note that when you ignore providers for a specific request, the list of ignored providers is merged with your account-wide ignored providers.

</Tip>

### Example: Ignoring Azure for a request calling GPT-4 Omni

Here's an example that will ignore Azure for a request calling GPT-4 Omni:

<TSFetchCodeBlock
  uriPath='/api/v1/chat/completions'
  body={{
    model: 'openai/gpt-4o',
    messages: [{ role: 'user', content: 'Hello' }],
    provider: {
      ignore: ['Azure'],
    },
  }}
/>

## Quantization

Quantization reduces model size and computational requirements while aiming to preserve performance. Most LLMs today use FP16 or BF16 for training and inference, cutting memory requirements in half compared to FP32. Some optimizations use FP8 or quantization to reduce size further (e.g., INT8, INT4).

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `quantizations` | string[] | - | List of quantization levels to filter by (e.g. `["int4", "int8"]`). [Learn more](#quantization) |

<Warning>
  Quantized models may exhibit degraded performance for certain prompts,
  depending on the method used.
</Warning>

Providers can support various quantization levels for open-weight models.

### Quantization Levels

By default, requests are load-balanced across all available providers, ordered by price. To filter providers by quantization level, specify the `quantizations` field in the `provider` parameter with the following values:

- `int4`: Integer (4 bit)
- `int8`: Integer (8 bit)
- `fp4`: Floating point (4 bit)
- `fp6`: Floating point (6 bit)
- `fp8`: Floating point (8 bit)
- `fp16`: Floating point (16 bit)
- `bf16`: Brain floating point (16 bit)
- `fp32`: Floating point (32 bit)
- `unknown`: Unknown

### Example: Requesting FP8 Quantization

Here's an example that will only use providers that support FP8 quantization:

<TSFetchCodeBlock
  uriPath='/api/v1/chat/completions'
  body={{
    model: 'meta-llama/llama-3.1-8b-instruct',
    messages: [{ role: 'user', content: 'Hello' }],
    provider: {
      quantizations: ['fp8'],
    },
  }}
/>

## Terms of Service

You can view the terms of service for each provider below. You may not violate the terms of service or policies of third-party providers that power the models on OpenRouter.

- `OpenAI`: https://openai.com/policies/row-terms-of-use/
- `Anthropic`: https://www.anthropic.com/legal/commercial-terms
- `Google Vertex`: https://cloud.google.com/terms/
- `Google AI Studio`: https://cloud.google.com/terms/
- `Amazon Bedrock`: https://aws.amazon.com/service-terms/
- `Groq`: https://groq.com/terms-of-use/
- `SambaNova`: https://sambanova.ai/terms-and-conditions
- `Cohere`: https://cohere.com/terms-of-use
- `Mistral`: https://mistral.ai/terms/#terms-of-use
- `Together`: https://www.together.ai/terms-of-service
- `Together (lite)`: https://www.together.ai/terms-of-service
- `Fireworks`: https://fireworks.ai/terms-of-service
- `DeepInfra`: https://deepinfra.com/docs/data
- `Lepton`: https://www.lepton.ai/policies/tos
- `NovitaAI`: https://novita.ai/legal/terms-of-service
- `Avian.io`: https://avian.io/privacy
- `Lambda`: https://lambdalabs.com/legal/privacy-policy
- `Azure`: https://www.microsoft.com/en-us/legal/terms-of-use?oneroute=true
- `Modal`: https://modal.com/legal/terms
- `AnyScale`: https://www.anyscale.com/terms
- `Replicate`: https://replicate.com/terms
- `Perplexity`: https://www.perplexity.ai/hub/legal/perplexity-api-terms-of-service
- `Recursal`: https://featherless.ai/terms
- `OctoAI`: https://octo.ai/docs/faqs/privacy-and-security
- `DeepSeek`: https://chat.deepseek.com/downloads/DeepSeek%20Terms%20of%20Use.html
- `Infermatic`: https://infermatic.ai/privacy-policy/
- `AI21`: https://studio.ai21.com/privacy-policy
- `Featherless`: https://featherless.ai/terms
- `Inflection`: https://developers.inflection.ai/tos
- `xAI`: https://x.ai/legal/terms-of-service
- `Cloudflare`: https://www.cloudflare.com/service-specific-terms-developer-platform/#developer-platform-terms
- `SF Compute`: https://inference.sfcompute.com/privacy
- `Minimax`: https://intl.minimaxi.com/protocol/terms-of-service
- `Nineteen`: https://nineteen.ai/tos
- `Liquid`: https://www.liquid.ai/terms-conditions
- `nCompass`: https://ncompass.tech/terms
- `inference.net`: https://inference.net/terms
- `Friendli`: https://friendli.ai/terms-of-service
- `AionLabs`: https://www.aionlabs.ai/terms/
- `Alibaba`: https://www.alibabacloud.com/help/en/legal/latest/alibaba-cloud-international-website-product-terms-of-service-v-3-8-0
- `Nebius AI Studio`: https://docs.nebius.com/legal/studio/terms-of-use/
- `Chutes`: https://chutes.ai/tos
- `kluster.ai`: https://www.kluster.ai/terms-of-use
- `Crusoe`: https://legal.crusoe.ai/open-router#managed-inference-tos-open-router
- `Targon`: https://targon.com/terms
- `Ubicloud`: https://www.ubicloud.com/docs/about/terms-of-service
- `Parasail`: https://www.parasail.io/legal/terms
- `Phala`: https://red-pill.ai/terms
- `Cent-ML`: https://centml.ai/terms-of-service/
- `Venice`: https://venice.ai/terms
- `OpenInference`: https://www.openinference.xyz/terms
- `Atoma`: https://atoma.network/terms
- `01.AI`: https://platform.01.ai/privacypolicy
- `HuggingFace`: https://huggingface.co/terms-of-service
- `Mancer`: https://mancer.tech/terms
- `Mancer (private)`: https://mancer.tech/terms
- `Hyperbolic`: https://hyperbolic.xyz/privacy
- `Hyperbolic (quantized)`: https://hyperbolic.xyz/privacy
- `Lynn`: https://api.lynn.app/policy

## JSON Schema for Provider Preferences

For a complete list of options, see this JSON schema:

<ZodToJSONSchemaBlock
  title='Provider Preferences Schema'
  schema={ProviderPreferencesSchema}
/>
