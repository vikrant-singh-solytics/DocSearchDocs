# DocSearch — Design Docs

DocSearch is an AI-powered document intelligence platform: multi-tenant document ingestion, RAG-based chat, compliance assessment, regulatory mapping, and NLP-to-structured-query features. Built on **Django 4.2 + Django REST Framework**, backed by **Milvus** for vector storage, **LangChain** for LLM orchestration, and **Celery** for async processing.

This site documents two things:

1. **How the platform is designed** — request lifecycle, auth, the LLM/embeddings layer, VectorDB abstraction, feature modules, async processing, and configuration.
2. **How to add new features** — a strict contract every new endpoint must follow. See [Feature Guide → The contract](feature-guide/contract.md).

## Start here

| If you are… | Read… |
|---|---|
| New to the codebase | [Getting Started](getting-started.md) → [Architecture overview](architecture/overview.md) |
| Adding a new API endpoint | [The contract](feature-guide/contract.md) → [Worked example](feature-guide/walkthrough.md) |
| Writing performant AI code | [Good practices](feature-guide/good-practices.md) |
| Reviewing someone else's PR | [Review checklist](feature-guide/review-checklist.md) |
| Debugging a request | [Request lifecycle](architecture/request-lifecycle.md) |
| Understanding the LLM layer | [LLM & Embeddings](architecture/llm-and-embeddings.md) |
| Understanding vector storage | [VectorDB (Milvus)](architecture/vectordb.md) |
| Understanding a feature module | [Features overview](features/overview.md) |
| Feeding the docs to an LLM | [LLM manifest](llms.txt) |

## Core architectural rule

> **No view ever calls an LLM, queries Milvus, or accesses the ORM directly.** Each feature has a dedicated `run_query.py` (or equivalent module) that owns all AI logic. Views validate the request and call into that module. Models are only touched through `Application/models.py`.

Everything else in this site enforces that rule. See [Data layer](architecture/data-layer.md) and [The contract](feature-guide/contract.md).

## Repository layout snapshot

```
DocSearch/
├── DocSearch/               # Django project (settings, urls, celery, wsgi/asgi)
├── Application/             # Core Django app: models, views, serializers, auth, urls
│   ├── views/               # DRF views (one module per feature)
│   ├── serializers/         # DRF serializers (mirrors views/)
│   ├── migrations/          # Django ORM migrations
│   ├── models.py            # Document, Session, ChatBox, Tenant
│   ├── auth.py              # API key auth (IsAdminAPIKeyValid, IsCompanyAPIKeyValid)
│   └── urls.py              # URL registration for all endpoints
├── Query/                   # RAG query runner (cross-encoder reranking)
├── AutoTag/                 # LLM-powered document tagging
├── DocAssess/               # Document assessment pipeline (V1 + V2)
├── RegulatoryMapping/       # Regulatory compliance mapping (V1)
├── RegulatoryMappingV2/     # Regulatory compliance mapping (V2, decompose-evaluate)
├── SuggestFields/           # AI field suggestion from documents
├── TextToSQL/               # Natural language → SQL
├── TextToFilter/            # Natural language → filter expressions
├── TextToWorkflow/          # Natural language → workflow JSON
├── VectorDB/                # Milvus abstraction (VectorDBProvider base + MilvusDB)
├── LLM/                     # LLM provider factory (Bedrock / OpenAI / Anthropic)
├── Embeddings/              # Embedding provider factory
├── Prompts/                 # Shared prompt templates
├── Tools/                   # Context-formatting tools (model data, schema)
├── Constants/               # TypeOfSearch enum, shared utilities
├── Middleware/               # Logstash logging middleware
├── DocSearchConfig.json     # Runtime config (NOT committed)
├── docs/                    # ← you are here
└── mkdocs.yml
```
