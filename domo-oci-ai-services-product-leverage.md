# Domo + OCI AI Services: Feature Parity and Product Expansion

This document describes how Domo can leverage Oracle Cloud Infrastructure (OCI) managed AI services across the product suite. It is organized into two tracks:

1) Match existing capabilities (parity with current managed AI patterns such as AWS Bedrock or GCP Vertex usage).
2) Add new, OCI-enabled features (incremental product value once parity is achieved).

The intent is to give Domo engineering and product teams a practical menu of options, plus a clear implementation pattern that keeps the platform secure and multi-tenant safe.

## Executive summary

Domo can treat OCI as:

- A managed foundation model (FM) inference platform via OCI Generative AI (Bedrock-like).
- A broader ML platform via OCI Data Science (Vertex AI / SageMaker-like).
- A set of prebuilt AI APIs via OCI AI Services (Language, Vision, Speech, Document Understanding, etc.).

The most successful approach is to build a thin, Domo-managed “AI Gateway” layer that:

- Abstracts providers (OCI Generative AI now; optional multi-cloud later).
- Enforces governance (tenant isolation, PII handling, policy guardrails).
- Centralizes observability (prompt logs, cost controls, evaluation, A/B testing).
- Provides stable internal APIs to Domo product teams.

## OCI AI building blocks (what to use, when)

### OCI Generative AI (managed LLMs and embeddings)

Best for:

- Chat and assistants (conversational UX, tool use, action planning).
- Text generation (SQL generation, explanations, narratives).
- Summarization (dashboards, alerts, tickets, docs).
- Embeddings (semantic search, RAG, recommendations).

Operational options:

- Shared endpoints (fastest to start).
- Dedicated AI Cluster + Private Endpoint (stronger isolation, private networking, regulated customers).

Model availability is region-dependent and changes over time; verify in the OCI model catalog. For current OCI Generative AI managed model IDs + lifecycle dates, see `domo-oci-genai-model-catalog.md`.

### OCI AI Services (prebuilt AI APIs)

Best for:

- Language: entity extraction, key phrases, sentiment, classification, translation, moderation-style enrichment.
- Document Understanding: OCR + structured extraction from PDFs/scans (receipts, invoices, forms).
- Vision: image classification and detection use cases.
- Speech: speech-to-text (voice commands, meeting/audio ingestion).

These services are useful where a deterministic API is preferred over prompt-based LLM behavior.

### OCI Data Science (ML platform)

Best for:

- Custom model training (forecasting, anomaly detection, churn/propensity, classification).
- Pipelines and jobs for batch scoring or offline evaluation.
- Model deployments for non-LLM models (or specialized models Domo owns).

This is the OCI equivalent of “bring your own model + MLOps” workflows.

### RAG and vector storage options on OCI

Most Domo assistants benefit from Retrieval-Augmented Generation (RAG) so answers reflect tenant data, governance rules, and current metadata. On OCI, common building blocks are:

- Object Storage: source documents, prompt templates, evaluation sets, and cached artifacts.
- Vector store:
  - OCI Search with OpenSearch (managed OpenSearch) for search + vector retrieval, or
  - Oracle Database 23ai vector search (ADB/DB 23ai options) for vectors close to metadata and structured data.

## Part 1: Match existing Domo AI functionality on OCI

This section focuses on “no-new-feature” replacements: moving current managed AI dependencies to OCI equivalents and making the platform production-ready.

### 1) Chat-based analytics (Domo AI assistant / “chat with my data”)

What users expect today:

- Natural language questions that return accurate results grounded in Domo datasets and dashboards.
- Explanations, summaries, and follow-ups.
- Safe behavior aligned with tenant permissions and governance.

OCI implementation:

- Use OCI Generative AI for chat + tool planning.
- Use embeddings + vector retrieval to ground the model on:
  - dataset schemas (tables, columns, data types, synonyms),
  - metric definitions (Beast Modes / calculated fields),
  - dashboard/card metadata and filters,
  - lineage, ownership, and business glossary content,
  - curated knowledge content (runbooks, wiki, metric catalogs).
- Enforce multi-tenant isolation by:
  - separating indexes per tenant (or per tenant partition) and filtering at retrieval time, and
  - scoping retrieval to the requesting user’s entitlements.

Where OCI AI Services can help:

- Language entity extraction and classification can “pre-parse” user intent (metric vs dimension vs time filter) before LLM execution.

### 2) NLQ-to-SQL (or NLQ-to-Domo query plan)

What users expect today:

- A natural language query becomes a safe, correct query plan (SQL or internal query language).
- Guardrails to prevent data leakage, expensive scans, or unsafe operations.

OCI implementation:

- Use OCI Generative AI for:
  - query translation (NL -> SQL/query-plan),
  - query explanation and debugging,
  - “repair” loops (if query fails, revise).
- Add deterministic checks outside the model:
  - schema/permission validation,
  - safe-list allowed operations,
  - row-level security enforcement,
  - cost guards (timeouts, row limits, sampling).

### 3) Summaries and narratives (dashboards, alerts, “what changed?”)

What users expect today:

- A human-readable summary of a dashboard, a KPI change, or an alert.
- Root-cause hints and next questions.

OCI implementation:

- Use OCI Generative AI summarization or chat models.
- Provide the model with curated context:
  - “KPI packet” (metric definition, baseline, segment deltas, time comparisons),
  - change-point/anomaly metadata from Domo’s analytics engine,
  - runbook links and owner/contact metadata.

Where OCI Data Science can help:

- Train/host anomaly detection or forecasting models for consistent scoring across tenants.

### 4) Semantic search across Domo assets (datasets, dashboards, cards, docs)

What users expect today:

- “Find the dashboard about churn” or “show me the dataset that has net revenue retention”.

OCI implementation:

- Embeddings via OCI Generative AI.
- Store vectors in OCI OpenSearch or Oracle DB vector search.
- Rank and filter results by tenant, entitlement, recency, usage, and certification status.

### 5) Document ingestion and extraction (receipts, invoices, contracts, PDFs)

What users expect today:

- Upload documents and turn them into structured datasets.
- Extract key fields reliably (not “best effort”).

OCI implementation:

- Use OCI Document Understanding for OCR + structured extraction.
- Use OCI Vision for image-related enrichment (when applicable).
- Use OCI Generative AI for:
  - normalization (convert extracted fields into canonical schema),
  - classification/routing (choose the right extractor),
  - narrative summaries (“what does this contract say?”).

### 6) Text enrichment for datasets (tags, topics, sentiment, translation)

What users expect today:

- Enrich text columns and customer feedback at scale.

OCI implementation:

- Use OCI Language for:
  - entity extraction, key phrases, sentiment, classification, translation (where supported).
- Use OCI Generative AI for:
  - flexible classification when rules and deterministic models are insufficient,
  - taxonomy generation, topic discovery, and “explain the trend”.

### 7) Voice features (mobile, accessibility, meeting/audio ingestion)

What users expect today:

- “Ask Domo” with voice, or upload audio and search it.

OCI implementation:

- Use OCI Speech for speech-to-text.
- Feed transcripts into:
  - OCI Generative AI for summarization and actions,
  - embeddings for search (“find the meeting where churn was discussed”).

### 8) Domo developer and admin assistance (internal parity)

What teams expect today:

- Faster workflows for creating connectors, ETL jobs, cards, and governance policies.

OCI implementation:

- Use OCI Generative AI to provide:
  - code and configuration assistance (connector templates, API usage, SQL snippets),
  - admin help (policy explanations, least-privilege suggestions),
  - migration assistance (AWS -> OCI mappings).

## Part 2: New OCI-enabled features Domo can add

Once parity is stable, OCI capabilities can support higher-leverage product features. These features are additive and can be rolled out progressively by tenant, region, or SKU.

### 1) “Dataset copilot” for Magic ETL and data prep

New capabilities:

- Generate or edit ETL pipelines from natural language.
- Recommend joins, deduping rules, type casting, and data quality checks.
- Explain why a pipeline fails and propose repairs.

OCI implementation:

- Use OCI Generative AI for planning and code generation.
- Use OCI Data Science jobs for offline evaluation:
  - validate that generated pipelines meet invariants,
  - measure drift and correctness across real customer datasets.

### 2) “Dashboard copilot” (build, refactor, and explain dashboards)

New capabilities:

- Create a dashboard from a goal statement (“Build an exec view for ARR and churn”).
- Refactor cards for performance and clarity.
- Generate a narrative “story” view and suggest next questions.

OCI implementation:

- Use OCI Generative AI for:
  - chart selection recommendations,
  - filter and segmentation suggestions,
  - narrative generation.
- Ground the model using RAG over:
  - metric catalog,
  - approved chart patterns,
  - tenant-specific glossary.

### 3) Governed RAG with automatic policy-aware citations

New capabilities:

- The assistant cites specific datasets/cards/docs that support an answer.
- Answers are automatically scoped by row-level security and certification status.

OCI implementation:

- Use embeddings + retrieval filters tied to Domo’s entitlement graph.
- Store citations as first-class objects (for auditability).

### 4) Automated data governance (classification, PII detection, retention hints)

New capabilities:

- Detect sensitive columns and recommend policies.
- Flag potential compliance risk in new datasets.
- Suggest retention and masking rules.

OCI implementation:

- Use OCI Language + OCI Generative AI for classification and policy suggestions.
- Log decisions and changes via OCI Audit/Logging and Domo’s governance audit trail.

### 5) Proactive “insight stream” (anomaly -> explanation -> action)

New capabilities:

- Detect anomalies, explain likely causes, recommend follow-up analysis, and suggest actions (create alert, open ticket, notify owner).

OCI implementation:

- Use OCI Data Science (or deterministic analytics) for anomaly detection.
- Use OCI Generative AI for explanation, hypothesis generation, and action planning.

### 6) Multi-modal analytics (images + text + voice)

New capabilities:

- Ask questions about images embedded in reports (screenshots, charts, slides).
- Convert photos/scans to datasets and dashboards.
- Voice-driven navigation and insights.

OCI implementation:

- OCI Vision + Document Understanding for extraction.
- OCI Speech for transcription.
- OCI Generative AI for synthesis and “next step” actions.

### 7) Enterprise “AI operations” inside Domo (for support and SRE)

New capabilities:

- Log and incident summarization.
- Runbook-driven troubleshooting assistants.
- Change impact analysis (“what services will be affected if we deploy X?”).

OCI implementation:

- Ingest logs/tickets/runbooks into a secured knowledge base.
- Use RAG + tool use to recommend actions and link to evidence.

## Reference architecture pattern: Domo AI Gateway on OCI

To keep product teams unblocked and keep governance consistent, treat OCI AI services as “shared platform services” behind a Domo-managed gateway.

Recommended responsibilities of the AI Gateway:

- Provider abstraction: OCI Generative AI now; optional multi-provider later.
- Routing: choose model per task (chat vs summarization vs embedding).
- Prompt templates: versioned prompts per feature and per SKU.
- Safety and governance:
  - tenant isolation and entitlement enforcement,
  - PII/secret redaction before model calls,
  - allow/deny lists for tools and actions,
  - response filtering and citation requirements where needed.
- Observability:
  - structured logging (prompt metadata, latency, token usage, cost estimates),
  - tracing for request -> tool -> query -> answer paths,
  - evaluation hooks and A/B experiment flags.

Deployment notes:

- Run AI Gateway on OKE or OCI Compute/Container Instances.
- Prefer OCI-native identity (instance principals / workload identity / resource principals) over long-lived API keys.
- For regulated tenants, use Private Endpoint and/or Dedicated AI Cluster and keep traffic on private OCI networking.

## Security, privacy, and multi-tenancy considerations

Key principles:

- Never send raw data that the user is not entitled to see.
- Treat prompts, retrieved context, and model outputs as sensitive logs.
- Keep a clear audit trail of:
  - which data was retrieved,
  - which model was called,
  - which tools/actions executed,
  - what output was returned to the user.

Recommended controls:

- Compartment design:
  - separate “domo-ai” resources (endpoints, clusters, private endpoints) from general app resources,
  - use tags for cost and quota management.
- Least-privilege IAM:
  - inference-only permissions for application runtimes,
  - “manage” permissions only for platform administrators.
- Data handling:
  - store prompt templates and evaluation sets in Object Storage with tight IAM policies,
  - encrypt secrets with OCI Vault,
  - use private networking where possible.

## Practical rollout plan

1) Stand up OCI Generative AI access in a dedicated compartment and validate models in the Playground.
2) Implement AI Gateway with a minimal set of endpoints: `chat`, `summarize`, `embed`.
3) Integrate one production feature (assistant chat or summarization) behind feature flags.
4) Add RAG with vector storage (OpenSearch or DB vector search) and entitlement-aware retrieval.
5) Expand to AI Services (Document Understanding, Language, Speech) for deterministic enrichment workflows.
6) Add evaluation, regression tests, and cost/latency SLOs; then scale feature coverage.

## Appendix A: Service mapping (AWS/GCP -> OCI)

This is a typical mapping used for “parity” migrations:

| Common capability | AWS (example) | GCP (example) | OCI equivalent |
|---|---|---|---|
| Managed LLM inference | Bedrock | Vertex AI (GenAI) | OCI Generative AI |
| ML platform (custom models) | SageMaker | Vertex AI | OCI Data Science |
| Text analytics | Comprehend | Cloud Natural Language | OCI Language (AI Services) |
| OCR/doc extraction | Textract | Document AI | OCI Document Understanding |
| Image analysis | Rekognition | Vision AI | OCI Vision |
| Speech-to-text | Transcribe | Speech-to-Text | OCI Speech |
| Vector search | OpenSearch / Pinecone | Matching Engine / Vector DB | OCI OpenSearch or Oracle DB vector search |

## Appendix B: OCI IAM policy snippets (Generative AI)

Exact policy design depends on your compartment strategy, but the core resource types are consistent.

Administrative (broad):

```text
allow group <your-group-name> to manage generative-ai-family in compartment <your-compartment-name>
```

Inference-only (recommended for app runtimes):

```text
allow group <your-group-name> to use generative-ai-chat in compartment <your-compartment-name>
allow group <your-group-name> to use generative-ai-text-generation in compartment <your-compartment-name>
allow group <your-group-name> to use generative-ai-text-summarization in compartment <your-compartment-name>
allow group <your-group-name> to use generative-ai-text-embedding in compartment <your-compartment-name>
```

Managing endpoints/clusters (platform administrators):

```text
allow group <your-group-name> to manage generative-ai-endpoint in compartment <your-compartment-name>
allow group <your-group-name> to manage generative-ai-dedicated-ai-cluster in compartment <your-compartment-name>
allow group <your-group-name> to manage generative-ai-private-endpoint in compartment <your-compartment-name>
allow group <your-group-name> to read generative-ai-work-request in compartment <your-compartment-name>
```

If you support fine-tuning datasets stored in Object Storage:

```text
allow group <your-group-name> to manage object-family in compartment <compartment-with-bucket>
allow group <your-group-name> to use object-family in compartment <compartment-with-bucket>
```

## Appendix C: Notes on model availability (including Gemini)

- OCI Generative AI model catalogs are region-dependent; verify availability in the OCI Console.
- Google Gemini models are available via OCI Generative AI in some regions/serving modes; verify in the OCI model catalog and `domo-oci-genai-model-catalog.md`. If a specific Gemini model isn't available in the target region, call Vertex AI cross-cloud behind the same Domo AI Gateway abstraction.
