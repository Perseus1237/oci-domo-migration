# OCI Generative AI: Managed LLM Model Catalog + Lifecycle Roadmap

Last updated: 2026-01-29

Sources:
- https://docs.public.oneportal.content.oci.oraclecloud.com/en-us/iaas/Content/generative-ai/pretrained-models.htm
- https://docs.public.oneportal.content.oci.oraclecloud.com/en-us/iaas/Content/generative-ai/model-endpoint-regions.htm
- https://docs.public.oneportal.content.oci.oraclecloud.com/en-us/iaas/Content/generative-ai/deprecating.htm

Notes:
- Model availability is region-dependent and changes over time; validate in the OCI Console and the “Models by Region” documentation.
- The retirement columns are sourced from each model’s “Release Date” section. If a retirement date is in the past, that serving mode is retired.
- Some entries have multiple API model IDs (for example, Grok “Fast” variants).

## Managed LLM models (chat + text generation)

| Provider | Model | Model ID | Type | Modality | Mode | Status | Release | On-demand retirement | Dedicated retirement |
|---|---|---|---|---|---|---|---|---|---|
| cohere | Cohere Command A | `cohere.command-a-03-2025` | Chat | Text | On-demand + Dedicated | Available | 2025-05-14 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| cohere | Cohere Command A Reasoning | `cohere.command-a-reasoning-08-2025` | Chat | Text | On-demand + Dedicated | Available | 2026-01-21 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| cohere | Cohere Command A Vision | `cohere.command-a-vision-07-2025` | Chat | Multimodal | On-demand + Dedicated | Available | 2026-01-21 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| cohere | Cohere Command R (08-2024) | `cohere.command-r-08-2024` | Chat | Text | On-demand + Dedicated | Available | 2024-11-14 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| cohere | Cohere Command R (Retired) | `cohere.command-r-16k` | Chat | Text | On-demand + Dedicated | Retired | 2024-06-04 | 2025-01-16 | 2025-08-07 |
| cohere | Cohere Command R+ (08-2024) | `cohere.command-r-plus-08-2024` | Chat | Text | On-demand + Dedicated | Available | 2024-11-14 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| cohere | Cohere Command R+ (Retired) | `cohere.command-r-plus` | Chat | Text | On-demand + Dedicated | Retired | 2024-06-18 | 2025-01-16 | 2025-08-07 |
| google | Google Gemini 2.5 Flash | `google.gemini-2.5-flash` | Chat | Multimodal | On-demand | Available | 2025-10-01 | Tentative | This model isn't available for the dedicated mode. |
| google | Google Gemini 2.5 Flash-Lite | `google.gemini-2.5-flash-lite` | Chat | Multimodal | On-demand | Available | 2025-10-01 | Tentative | This model isn't available for the dedicated mode. |
| google | Google Gemini 2.5 Pro | `google.gemini-2.5-pro` | Chat | Multimodal | On-demand | Available | 2025-10-01 | Tentative | This model isn't available for the dedicated mode. |
| meta | Meta Llama 3 (70B) | `meta.llama-3-70b-instruct` | Chat | Text | On-demand + Dedicated | Retired | 2024-06-04 | 2024-11-12 | 2025-08-07 |
| meta | Meta Llama 3.1 (405B) | `meta.llama-3.1-405b-instruct` | Chat | Text | On-demand + Dedicated | Available | 2024-09-19 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| meta | Meta Llama 3.1 (70B) (Retired) | `meta.llama-3.1-70b-instruct` | Chat | Text | On-demand + Dedicated | Retired | 2024-09-19 | 2025-07-10 | 2025-08-07 |
| meta | Meta Llama 3.2 11B Vision | `meta.llama-3.2-11b-vision-instruct` | Chat | Multimodal | Dedicated | Available | 2024-11-14 | On-demand mode is not available for this model. | At least 6 months after the release of the 1st replacement model. |
| meta | Meta Llama 3.2 90B Vision | `meta.llama-3.2-90b-vision-instruct` | Chat | Multimodal | On-demand + Dedicated | Available | 2024-11-14 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| meta | Meta Llama 3.3 (70B) | `meta.llama-3.3-70b-instruct` | Chat | Text | On-demand + Dedicated | Available | 2025-02-07 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| meta | Meta Llama 4 Maverick | `meta.llama-4-maverick-17b-128e-instruct-fp8` | Chat | Multimodal | On-demand + Dedicated | Available | 2025-05-14 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| meta | Meta Llama 4 Scout | `meta.llama-4-scout-17b-16e-instruct` | Chat | Multimodal | On-demand + Dedicated | Available | 2025-05-14 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| openai | OpenAI gpt-oss-120b | `openai.gpt-oss-120b` | Chat | Text | On-demand | Available | 2025-11-17 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| openai | OpenAI gpt-oss-20b | `openai.gpt-oss-20b` | Chat | Text | On-demand | Available | 2025-11-17 | At least one month after the release of the 1st replacement model. | At least 6 months after the release of the 1st replacement model. |
| xai | xAI Grok 3 | `xai.grok-3` | Chat | Text | On-demand | Available | 2025-06-24 | Tentative | This model isn't available for the dedicated mode. |
| xai | xAI Grok 3 Fast | `xai.grok-3-fast` | Chat | Text | On-demand | Available | 2025-06-24 | Tentative | This model isn't available for the dedicated mode. |
| xai | xAI Grok 3 Mini | `xai.grok-3-mini` | Chat | Text | On-demand | Available | 2025-06-24 | Tentative | This model isn't available for the dedicated mode. |
| xai | xAI Grok 3 Mini Fast | `xai.grok-3-mini-fast` | Chat | Text | On-demand | Available | 2025-06-24 | Tentative | This model isn't available for the dedicated mode. |
| xai | xAI Grok 4 | `xai.grok-4` | Chat | Text | On-demand | Available | 2025-07-23 | Tentative | This model isn't available for the dedicated mode. |
| xai | xAI Grok 4 Fast (non-reasoning) | `xai.grok-4-fast-non-reasoning` | Chat | Text | On-demand | Available | 2025-10-10 | Tentative | This model isn't available for the dedicated mode. |
| xai | xAI Grok 4 Fast (reasoning) | `xai.grok-4-fast-reasoning` | Chat | Text | On-demand | Available | 2025-10-10 | Tentative | This model isn't available for the dedicated mode. |
| xai | xAI Grok 4.1 Fast (non-reasoning) | `xai.grok-4.1-fast-non-reasoning` | Chat | Text | On-demand | Available | 2026-01-21 | Tentative | This model isn't available for the dedicated mode. |
| xai | xAI Grok 4.1 Fast (reasoning) | `xai.grok-4.1-fast-reasoning` | Chat | Text | On-demand | Available | 2026-01-21 | Tentative | This model isn't available for the dedicated mode. |
| xai | xAI Grok Code Fast 1 | `xai.grok-code-fast-1` | Chat | Text | On-demand | Available | 2025-10-01 | Tentative | This model isn't available for the dedicated mode. |
| cohere | Cohere Command (52B) | `cohere.command` | Text Generation / Summarization | Text | On-demand + Dedicated | Retired | 2024-02-07 | 2024-10-02 | 2025-08-07 |
| cohere | Cohere Command Light | `cohere.command-light` | Text Generation / Summarization | Text | On-demand + Dedicated | Retired | 2024-02-07 | 2024-10-02 | 2025-08-07 |

## Roadmap guidance (how Domo should manage model churn)

- Treat “model selection” as a platform concern (route by task: chat vs reasoning vs multimodal vs cost/latency).
- Keep OCI model IDs behind an internal `AI Gateway` abstraction so teams can swap models without app changes.
- Track the OCI retirement schedule and run prompt regression tests before switching defaults.
- For regulated customers, prefer private networking and dedicated hosting where supported.
