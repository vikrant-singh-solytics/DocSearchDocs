# Document Chat (RAG)

**Endpoint:** `POST /chat/`  
**View:** `Application/views/Chat.py::DocChatClass`  
**Runner:** `Query/run_query.py::answer_docs()`  
**Auth:** `IsCompanyAPIKeyValid`

## What it does

Answers natural-language questions over documents stored in Milvus. Maintains a multi-turn chat history per session. Uses a cross-encoder reranker for higher-quality retrieval.

## Request

```json
{
  "query": "What are the key risk factors?",
  "header": "<tenant-api-key>",
  "session_id": "<uuid>",
  "document_id": "<uuid>",
  "context_refresh": false
}
```

| Field | Required | Description |
|---|---|---|
| `query` | Yes | The user's question |
| `header` | Yes | Tenant API key (determines the Milvus collection) |
| `session_id` | Yes | Links this turn to a `Session` in the DB |
| `document_id` | No | Scopes the search to a single document |
| `context_refresh` | No | `true` to ignore prior chat history |

## Response

```json
{
  "answer": "The document identifies three primary risk factors: ...",
  "session_id": "<uuid>"
}
```

## Internal pipeline

```
1. Load Session + ChatBox history from DB
2. Apply token-budget suppression to history (_suppress_segment)
3. Build chat history list (_build_chat_history)
4. Search Milvus → top-K candidate chunks
5. Rerank with cross-encoder (_rerank → _get_cross_encoder)
6. Optional: keyword boost (_keyword_boost)
7. LLM call with context + history
8. Save new ChatBox turn to DB
9. Return answer
```

### Token-budget suppression

`_suppress_segment()` applies budget management to stored `ChatBox` entries. When the accumulated history approaches the LLM context limit, older turns have their `result_content` replaced with a summary token. `_token_estimate()` uses `tiktoken` for accurate counts.

### Reranking

`Query/run_query.py::_rerank()` loads a cross-encoder (via `_get_cross_encoder()`, singleton-loaded once per process) and rescores the top-K Milvus results. This two-stage retrieval pattern (vector search → cross-encoder rerank) significantly improves answer quality over raw vector similarity alone.

## Key files

| File | Role |
|---|---|
| `Application/views/Chat.py` | View, auth, history management |
| `Application/serializer/Chat.py::DocChatSerializers` | Request validation |
| `Query/run_query.py` | RAG pipeline, reranking |
| `Query/Utilities.py::create_search_expresion()` | Milvus filter expression builder |
| `Application/Utilities/RAGCommon.py` | Collection naming, response formatting |

## Document upload

Before chat can work, a document must be uploaded and embedded:

**Endpoint:** `POST /document/`  
**View:** `Application/views/Document.py::DocumentClass`

The upload flow:
1. Serializer validates the multipart form (file + metadata)
2. View calls `Application/Utilities/Document.py::extract_text_from_file()` to extract text (PDF via `pdfplumber`, DOCX via `unstructured`, Excel via `openpyxl`)
3. Text is chunked and embedded via `my_embedding()`
4. Chunks are stored in Milvus via `VectorDbManager.add_files()`
5. A `Document` record is saved to PostgreSQL

Other document endpoints: `GET /document/` (check existence via `CheckDocExistenceClass`), `DELETE /document/` (`DocumentClass.delete()`), `POST /attribute-sync/` (`AttributeSyncClass`).
