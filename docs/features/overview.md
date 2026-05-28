# Features — Overview

DocSearch ships nine AI-powered feature modules. Each is self-contained: its own folder, `run_query.py`, serializer, view, prompt, and `TypeOfSearch` namespace.

## Feature map

| Feature | Folder | Entry point | What it does |
|---|---|---|---|
| **Document Chat (RAG)** | `Query/` + `Application/views/Chat.py` | `Query/run_query.py::answer_docs()` | RAG-based Q&A over uploaded documents |
| **Document Assessment V1** | `DocAssess/` | `DocAssess/query.py::assess_doc()` | Evaluates a document against test prompts |
| **Document Assessment V2** | `DocAssess/` | `DocAssess/assess_v2.py::assess_doc_v2()` | Chunked, multi-prompt assessment pipeline |
| **Regulatory Mapping V1** | `RegulatoryMapping/` | `RegulatoryMapping/run_query.py` | Maps documents against regulatory guidelines |
| **Regulatory Mapping V2** | `RegulatoryMappingV2/` | `RegulatoryMappingV2/run_query.py` | Decompose-evaluate-summarise compliance pipeline |
| **AutoTag** | `AutoTag/` | `AutoTag/run_query.py::generate_tags()` | LLM-generated document tags |
| **SuggestFields** | `SuggestFields/` | `SuggestFields/run_query.py` | AI-powered form field extraction from documents |
| **Text-to-SQL** | `TextToSQL/` | `TextToSQL/Utilities.py::generate_sql_query()` | Natural language → SQL via schema embeddings |
| **Text-to-Filter** | `TextToFilter/` | `TextToFilter/run_query.py::text_to_filter()` | Natural language → structured filter expressions |
| **Text-to-Workflow** | `TextToWorkflow/` | `TextToWorkflow/run_query.py::generate_workflow_json()` | Natural language → workflow node/edge JSON |
| **Direct Model Chat** | `Application/views/ModelChatDirect.py` | `Tools/model_data.py` | Chat with a model entity using NimbusVault context |

## Shared infrastructure

All features share:

- **`my_llm()`** — single LLM factory with provider switching and token logging
- **`VectorDbManager`** — Milvus abstraction for vector storage/retrieval
- **`create_collection_name()`** — deterministic multi-tenant collection naming
- **`TypeOfSearch`** — enum that namespaces each feature's Milvus collection
- **`flatten_serializer_errors()`** — consistent 400 response format

## Detailed pages

- [Document Chat (RAG)](document-chat.md)
- [Document Assessment](document-assessment.md)
- [Regulatory Mapping](regulatory-mapping.md)
- [AutoTag](autotag.md)
- [SuggestFields](suggest-fields.md)
- [Text Intelligence (SQL / Filter / Workflow)](text-intelligence.md)
