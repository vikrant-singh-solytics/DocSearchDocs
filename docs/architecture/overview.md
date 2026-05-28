# Architecture Overview

DocSearch is a layered Django service where every HTTP request flows through the same pipeline and every feature module has the same internal shape. This page is the map.

## Layers, top to bottom

```
HTTP request
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Middleware stack                                                      │
│   LogstashLoggingMiddleware → CORS → allowed-CIDR                    │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ URL router (DocSearch/urls.py → Application/urls.py)                 │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ DRF views (Application/views/<Feature>.py)                           │
│   class-based, one method per HTTP verb                              │
│   API key auth (IsAdminAPIKeyValid / IsCompanyAPIKeyValid)            │
│   serializer validates request                                       │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Serializers (Application/serializer/<Feature>.py)                    │
│   input validation + response shaping                                │
└──────────────────────────────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Feature runner (<Feature>/run_query.py or equivalent)                │
│   all AI logic lives here: LLM calls, vector search, reranking       │
│   composes LLM/, VectorDB/, Embeddings/, Prompts/                    │
└──────────────────────────────────────────────────────────────────────┘
  │
  ├─► LLM/Providers.py        (my_llm() factory — Bedrock / OpenAI / Anthropic)
  ├─► Embeddings/Providers.py (my_embedding() factory — HuggingFace / OpenAI)
  ├─► VectorDB/VectorDB.py    (VectorDbManager — Milvus abstraction)
  └─► Application/models.py   (Django ORM — Document, Session, ChatBox, Tenant)
  │
  ▼
PostgreSQL  +  Milvus  +  Redis  +  AWS Bedrock / OpenAI / Anthropic
```

## Why this shape

| Constraint | What enforces it |
|---|---|
| Multi-tenant isolation | API key auth maps requests to a tenant; collection names in Milvus include the org/header key. |
| Auth everywhere | `IsAdminAPIKeyValid` / `IsCompanyAPIKeyValid` on every view. |
| LLM in one place | `LLM/Providers.py::my_llm()` is the only factory for LLM instances. Features import it — they never construct an LLM client directly. |
| Vector ops in one place | `VectorDB/VectorDB.py::VectorDbManager` wraps all Milvus calls. Feature modules call it; they do not import `pymilvus` directly. |
| Provider-switchable | `DocSearchConfig.json` selects `llm_provider`, `embeddings_provider`, `vectordb_provider` at startup — no code change to swap providers. |

## Where each thing lives

| Concern | Location |
|---|---|
| Settings + load `DocSearchConfig.json` | `DocSearch/settings.py` |
| Root URLs | `DocSearch/urls.py` |
| Feature URLs | `Application/urls.py` |
| Middleware | `Middleware/CustomLoggingMiddleware.py` |
| Celery app | `DocSearch/celery.py` |
| Views | `Application/views/<Feature>.py` |
| Serializers | `Application/serializer/<Feature>.py` |
| Auth | `Application/auth.py` |
| Models | `Application/models.py` |
| Document upload utilities | `Application/Utilities/Document.py` |
| RAG utilities | `Application/Utilities/RAGCommon.py` |
| LLM factory | `LLM/Providers.py` |
| LLM feature→model registry | `LLM/FeatureModels.py` |
| Embedding factory | `Embeddings/Providers.py` |
| VectorDB abstraction | `VectorDB/VectorDB.py` (manager), `VectorDB/Providers/` (base + Milvus impl) |
| Constants / search types | `Constants/constants.py` |
| Prompts | `Prompts/` (one module per feature) |
| Tools (context formatting) | `Tools/` |
| Feature runners | `<Feature>/run_query.py` |

Read the next pages in order if onboarding. Jump to [The contract](../feature-guide/contract.md) if you just need to ship an endpoint.
