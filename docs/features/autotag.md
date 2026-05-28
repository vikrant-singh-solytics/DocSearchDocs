# AutoTag

**Endpoint:** `POST /autotag/`  
**View:** `Application/views/AutoTag.py::AutoTagClass`  
**Runner:** `AutoTag/run_query.py::generate_tags()`  
**Auth:** `IsCompanyAPIKeyValid`

## What it does

Generates document tags using LLM analysis. Prioritises matching against a predefined tag vocabulary — if the document content aligns with known tags, those are preferred. Genuinely novel topics produce new tags.

## Request

```json
{
  "document_id": "<uuid>",
  "header": "<tenant-api-key>",
  "predefined_tags": ["Risk", "Compliance", "Credit Risk", "Liquidity"]
}
```

| Field | Required | Description |
|---|---|---|
| `document_id` | Yes | Document to tag |
| `header` | Yes | Tenant API key |
| `predefined_tags` | No | Preferred tag vocabulary to match against |

## Response

```json
{
  "tags": ["Credit Risk", "Regulatory Compliance", "Model Validation"]
}
```

## Internal pipeline

```
1. Retrieve document chunks from Milvus
2. Build prompt with predefined_tags list + document context
3. LLM call → tag list
4. Return tags
```

The prompt in `Prompts/AutoTag.py` instructs the LLM to:
1. First check whether the document matches any `predefined_tags`
2. Only generate new tags for content that genuinely doesn't match

This two-priority approach keeps tagging consistent across documents for the same organisation.

## Key files

| File | Role |
|---|---|
| `Application/views/AutoTag.py::AutoTagClass` | View + auth |
| `Application/serializer/AutoTag.py::AutoTagSerializer` | Request validation |
| `AutoTag/run_query.py::generate_tags()` | LLM tagging logic |
| `Prompts/AutoTag.py` | Tag generation prompt |
