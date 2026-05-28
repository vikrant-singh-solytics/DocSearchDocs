# Text Intelligence — SQL, Filter, Workflow

Three features translate natural language into structured outputs. All share the same architecture: a view → serializer → `run_query.py` → `my_llm()` pattern, with Milvus used for context retrieval where a schema is involved.

---

## Text-to-SQL

**Endpoints:**  
- `POST /text-to-sql/` — Milvus-backed (schema embeddings in vector store)
- `POST /text-to-sql/direct/` — Direct context (schema JSON passed in request)
- `POST /text-to-sql/sync-schema/` — Ingest table schema into Milvus

**View:** `Application/views/TextToSQL.py`  
**Runner:** `TextToSQL/Utilities.py::generate_sql_query()`  
**Auth:** `IsCompanyAPIKeyValid`

### What it does

Converts a natural-language question into a SQL query. Two retrieval modes:

| Mode | View | Context source |
|---|---|---|
| Milvus-backed | `TextToSQLView` | Retrieves relevant schema from Milvus, then generates SQL |
| Direct | `TextToSQLDirectView` | Schema JSON provided directly in the request |

`generate_sql_query()` (Milvus mode) and `generate_sql_query_direct()` (direct mode) both call `my_llm()` with `SQLPrompts` from `Prompts/TextToSql.py`.

`normalize_and_format_sql()` cleans and pretty-formats the LLM's SQL output before returning.

### Schema ingestion

`POST /text-to-sql/sync-schema/` (`SyncSchemaView`) ingests table schema:

```
TextToSQL/VectorDB.py::ingest_schema()
  ├── ingest_single_table()    — per-table ingestion
  ├── delete_table_schema()    — remove stale entries
  └── VectorDbManager.add_files()
```

`TextToFilter/Constants.py::format_entity_grouped_text()` + `format_attribute_text()` convert attribute schema dicts into text blocks for Milvus storage. These are shared between Text-to-SQL and Text-to-Filter.

### Streaming

`TextToSQLDirectView` supports streaming: `generate_sql_query_direct_stream()` yields tokens as they arrive from the LLM. The view uses Django's `StreamingHttpResponse`.

---

## Text-to-Filter

**Endpoint:** `POST /text-to-filter/`  
**View:** `Application/views/TextToFilter.py::TextToFilterView`  
**Runner:** `TextToFilter/run_query.py::text_to_filter()`  
**Auth:** `IsCompanyAPIKeyValid`

### What it does

Converts natural language queries like *"show me all high-risk models owned by the credit team"* into structured filter expressions for the NimbusVault UI.

### Pipeline

```
1. Retrieve entity/attribute schema from Milvus
2. _parse_schema_from_milvus()     — reconstruct schema dict from stored text blocks
3. _extract_sort_clause()          — parse ORDER BY intent
4. _remove_sort_clause()           — strip sort from rule-parsing input
5. _extract_having_advance_filters() — handle "having sub-process where..." syntax
6. _extract_basic_conditions()     — parse basic attribute-value filter rules
7. _normalize_ui_filter()          — map UI labels to backend field names
8. Return { "filters": [...], "sort": {...} }
```

### Key internal functions

`_extract_basic_conditions()` and `_extract_having_advance_filters()` are the most complex functions in the codebase (god node #5 with 12 edges). They handle:

- Compound conditions (`AND`, `OR`)
- Range queries (`between`, `more than`, `less than`)
- Multi-value lists
- Entity sub-process scoping (`having sub-process where ...`)

`_normalize_ui_filter()` (god node #3, 13 edges) resolves UI-friendly labels (e.g. "Risk Rating") to backend field names (e.g. `risk_rating`) using a lookup built from the schema. This is the final normalisation step before the filter is returned to the caller.

### Schema storage format

`TextToFilter/Constants.py` defines the text block formats used when ingesting schema into Milvus:

- `format_attribute_text()` — one text block per attribute (name, type, allowed values)
- `format_team_role_text()` — one text block per team role
- `format_entity_grouped_text()` — one document grouping all attributes for an entity type

---

## Text-to-Workflow

**Endpoint:** `POST /text-to-workflow/`  
**View:** `Application/views/TextToWorkflow.py::TextToWorkflowView`  
**Runner:** `TextToWorkflow/run_query.py::generate_workflow_json()`  
**Auth:** `IsCompanyAPIKeyValid`

### What it does

Converts a natural-language description into a workflow JSON with nodes and edges (for use in NimbusVault's workflow engine).

### Pipeline

```
1. LLM call with TextToWorkflow prompt
2. parse_llm_response()        — extract workflow data from LLM output
3. standardize_workflow_ids()  — normalise all node/edge IDs
4. validate_workflow()         — check structure meets requirements
5. Return workflow JSON { nodes: [...], edges: [...] }
```

`generate_workflow_json()` orchestrates all four steps, retrying if validation fails.

### `standardize_workflow_ids()`

Ensures all node and edge IDs are consistent (lowercase, snake_case, no spaces). This is necessary because the LLM may produce varied ID formats even when given a template.

### Validation

`validate_workflow()` checks:
- All edge `source` and `target` references point to existing node IDs
- Every node has a unique ID
- Required fields (`type`, `label`, `position`) are present on each node

---

## Key files summary

| Feature | Runner | Prompts | View |
|---|---|---|---|
| Text-to-SQL | `TextToSQL/Utilities.py` | `Prompts/TextToSql.py` | `Application/views/TextToSQL.py` |
| Text-to-Filter | `TextToFilter/run_query.py` | `Prompts/TextToFilter.py` | `Application/views/TextToFilter.py` |
| Text-to-Workflow | `TextToWorkflow/run_query.py` | `Prompts/TextToWorkflow.py` | `Application/views/TextToWorkflow.py` |
