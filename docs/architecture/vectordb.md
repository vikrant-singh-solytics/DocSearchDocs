# VectorDB (Milvus)

All vector storage operations go through a single abstraction layer. Feature modules never import `pymilvus` directly.

## Abstraction stack

```
Feature runner (run_query.py)
        │
        ▼
VectorDB/VectorDB.py :: VectorDbManager     ← public interface for all features
        │
        ▼
VectorDB/Providers/Milvus/Milvus.py :: MilvusDB   ← concrete Milvus implementation
        │
        ▼
VectorDB/Providers/Base.py :: VectorDBProvider     ← abstract base class
        │
        ▼
pymilvus  (Milvus 2.x)
```

## `VectorDbManager` — the interface

**File:** `VectorDB/VectorDB.py`

```python
from VectorDB.VectorDB import VectorDbManager

vdb = VectorDbManager()
vdb.initialize_db(collection_name, schema)
vdb.add_files(collection_name, texts, metadatas)
vdb.search(collection_name, query_text, top_k=5)
vdb.check_file_existence(collection_name, file_id)
vdb.delete_files(collection_name, file_ids)
vdb.delete_by_table(collection_name, table_name)
vdb.drop_collection(collection_name)
vdb.collection_exists(collection_name)
```

`VectorDbManager` reads `vectordb_provider` from `DocSearchConfig.json` and instantiates the right provider. Currently only `"Milvus"` is implemented.

## Collection naming

Collection names are deterministic and multi-tenant. They are always built via `create_collection_name()` in `Application/Utilities/RAGCommon.py`:

```python
from Application.Utilities.RAGCommon import create_collection_name
from Constants.constants import TypeOfSearch

name = create_collection_name(header=api_key, search_type=TypeOfSearch.DOCUMENT_SEARCH)
```

`TypeOfSearch` is the central enum (in `Constants/constants.py`) that namespaces collections per feature:

| `TypeOfSearch` value | Used by |
|---|---|
| `DOCUMENT_SEARCH` | Document Chat / RAG |
| `REGULATORY_MAPPING` | Regulatory Mapping V1 |
| `REGULATORY_MAPPING_V2` | Regulatory Mapping V2 |
| `TEXT_TO_SQL` | Text-to-SQL schema ingestion |
| `TEXT_TO_FILTER` | Text-to-Filter attribute storage |
| `SUGGEST_FIELDS` | SuggestFields |

This means every tenant × feature combination maps to its own isolated Milvus collection.

## Chunking

`MilvusDB._chunk_documents()` (in `VectorDB/Providers/Milvus/Milvus.py`) splits texts before ingestion using LangChain's `RecursiveCharacterTextSplitter`. Chunk size and overlap are configured per collection type.

## Metadata

`VectorDB/Providers/Milvus/Utilities.py::create_metadata()` standardises the metadata dict stored alongside each vector. Metadata fields enable:

- filtering by `file_id` for per-document operations
- filtering by `table_name` for Text-to-SQL schema management
- filtering by entity type for Text-to-Filter

## Admin endpoints

`Application/views/MilvusAdmin.py` exposes two management endpoints:

| Endpoint | View | Purpose |
|---|---|---|
| `GET /milvus/collections/` | `ListSchemaCollectionsView` | List all collections for a tenant |
| `DELETE /milvus/collections/<name>/` | `DeleteSchemaCollectionView` | Drop a collection |

These require `IsAdminAPIKeyValid`.

## When to use `drop_collection.py`

The repo root `drop_collection.py` is a standalone script for manually dropping a collection outside of the API. Use it when:

- Migrating to a new schema (must drop before re-ingesting)
- Cleaning up a corrupt or stale collection in dev

**Do not** drop collections in production without a re-ingestion plan — the feature that owns the collection will fail until data is restored.
