# SuggestFields

**Endpoint:** `POST /suggest-fields/`  
**View:** `Application/views/SuggestFields.py::SuggestFieldsClass`  
**Runner:** `SuggestFields/run_query.py`  
**Auth:** `IsCompanyAPIKeyValid`

## What it does

Extracts structured field values from document text using an LLM. Supports two modes:

- **Direct extraction** — used when this is the first or only document for an entity. The LLM reads the document text directly.
- **RAG-enhanced extraction** — used when prior documents exist for the same entity. Retrieves relevant chunks from Milvus before extracting, allowing the model to leverage historical context.

## Request

```json
{
  "document_id": "<uuid>",
  "header": "<tenant-api-key>",
  "attribute_schema": {
    "fields": [
      {
        "name": "model_name",
        "label": "Model Name",
        "type": "string",
        "allowed_values": null
      },
      {
        "name": "risk_rating",
        "label": "Risk Rating",
        "type": "string",
        "allowed_values": ["Low", "Medium", "High"]
      }
    ]
  },
  "entity_id": "<uuid>"
}
```

## Response

```json
{
  "suggestions": {
    "model_name": "Credit Scorecard v2",
    "risk_rating": "Medium"
  }
}
```

## Internal pipeline

```
1. Serializer validates attribute_schema (via validate_attribute_schema())
2. _format_schema_for_prompt() — compact JSON summary of field types + constraints
3. Check if prior documents exist for entity_id
   ├── No  → suggest_fields_direct()   — direct LLM extraction
   └── Yes → suggest_fields_rag()      — RAG-enhanced extraction
4. _merge_chunk_results()              — merge results from multiple chunks
5. Return suggestions dict
```

### `_format_schema_for_prompt()`

Converts the `attribute_schema` into a concise string the LLM can read. Calls `_extract_constraints()` to pull allowed values, min/max, and format hints from each field definition.

### `_merge_chunk_results()`

When processing multi-page documents, each chunk produces partial suggestions. This function merges them, keeping the highest-confidence value per attribute. Later chunks take precedence only if they provide a more complete answer.

### `_parse_llm_json()`

Parses JSON from LLM output, handling markdown code fences (` ```json ... ``` `) that some models emit even when instructed not to.

## Key files

| File | Role |
|---|---|
| `Application/views/SuggestFields.py` | View |
| `Application/serializer/SuggestFields.py::SuggestFieldsSerializer` | Validation including `validate_attribute_schema()` |
| `SuggestFields/run_query.py` | Extraction logic (direct + RAG modes) |
| `Prompts/SuggestFields.py` | Field extraction prompt |
