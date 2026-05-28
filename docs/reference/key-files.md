# Key Files

Quick reference for the files you'll touch most often.

## Configuration

| File | Purpose |
|---|---|
| `DocSearchConfig.json` | Runtime config — LLM provider, Milvus, Logstash, DocAssess paths. **Not committed.** |
| `DocSearch/settings.py` | Django settings. Loads `DocSearchConfig.json` at startup. |
| `requirements.txt` | Python dependencies. |
| `docker-compose.yml` | Local container topology. |

## Core platform

| File | Purpose |
|---|---|
| `DocSearch/urls.py` | Root URL conf — includes `Application/urls.py`. |
| `Application/urls.py` | All feature endpoint registrations. |
| `Application/auth.py` | `IsAdminAPIKeyValid`, `IsCompanyAPIKeyValid` — all auth logic. |
| `Application/models.py` | `Tenant`, `Document`, `Session`, `ChatBox` — the four Django models. |
| `Constants/constants.py` | `TypeOfSearch` enum, `flatten_serializer_errors()`, `Utilities`. |
| `Middleware/CustomLoggingMiddleware.py` | `LogstashLoggingMiddleware` — request/response logging. |

## LLM & Embeddings

| File | Purpose |
|---|---|
| `LLM/Providers.py` | `my_llm()` factory + `TokenUsageLogger`. The only place to construct an LLM. |
| `LLM/FeatureModels.py` | `FeatureModels` — maps features to Bedrock model IDs. `detect_from_stack()`. |
| `Embeddings/Providers.py` | `my_embedding()` factory. The only place to construct an embeddings client. |

## VectorDB

| File | Purpose |
|---|---|
| `VectorDB/VectorDB.py` | `VectorDbManager` — public interface for all vector operations. |
| `VectorDB/Providers/Milvus/Milvus.py` | `MilvusDB` — concrete Milvus implementation. |
| `VectorDB/Providers/Base.py` | `VectorDBProvider` — abstract base class. |
| `VectorDB/Providers/Milvus/Utilities.py` | `create_metadata()` — standardises Milvus metadata dicts. |

## Utilities

| File | Purpose |
|---|---|
| `Application/Utilities/Document.py` | `extract_text_from_file()` — PDF, DOCX, Excel, text extraction. |
| `Application/Utilities/RAGCommon.py` | `create_collection_name()`, `create_response()`. |
| `Query/Utilities.py` | `create_search_expresion()` — Milvus filter expression builder. |
| `TextToFilter/Constants.py` | Schema-to-text formatters for Milvus ingestion. |
| `Tools/model_data.py` | `format_model_context()` — context formatter for model chat. |
| `Tools/table_schema.py` | `format_schema_context()` — schema formatter for SQL/Filter. |

## Feature runners

| File | Feature |
|---|---|
| `Query/run_query.py` | Document Chat / RAG |
| `DocAssess/query.py` | DocAssess V1 |
| `DocAssess/assess_v2.py` | DocAssess V2 |
| `RegulatoryMapping/run_query.py` | Regulatory Mapping V1 |
| `RegulatoryMappingV2/run_query.py` | Regulatory Mapping V2 |
| `AutoTag/run_query.py` | AutoTag |
| `SuggestFields/run_query.py` | SuggestFields |
| `TextToSQL/Utilities.py` | Text-to-SQL |
| `TextToFilter/run_query.py` | Text-to-Filter |
| `TextToWorkflow/run_query.py` | Text-to-Workflow |
