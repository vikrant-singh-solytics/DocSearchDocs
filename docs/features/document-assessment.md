# Document Assessment

DocSearch ships two versions of document assessment. V1 is the simpler single-pass approach; V2 is a full chunked multi-prompt pipeline with configurable test cases.

---

## DocAssess V1

**Endpoint:** `GET /doc-assess/`  
**View:** `Application/views/DocAssess.py::DocAssessClass`  
**Runner:** `DocAssess/query.py::assess_doc()`  
**Auth:** `IsCompanyAPIKeyValid`

### What it does

Evaluates an uploaded document against a fixed set of prompts. Returns a pass/fail/partial result per prompt category.

### Request

```json
{
  "document_id": "<uuid>",
  "header": "<tenant-api-key>"
}
```

### Internal pipeline

1. Retrieve document chunks from Milvus
2. Call `my_llm()` with each assessment prompt
3. Aggregate results → return structured JSON

---

## DocAssess V2

**Endpoint:** `POST /doc-assess-v2/`  
**View:** `Application/views/DocAssessV2.py::DocAssessV2Class`  
**Runner:** `DocAssess/assess_v2.py::assess_doc_v2()`  
**Auth:** `IsCompanyAPIKeyValid`

### What it does

A comprehensive multi-prompt evaluation pipeline that:

1. Extracts full document text (no truncation)
2. Chunks text into overlapping token-based segments
3. Runs multiple prompt categories against each chunk in parallel
4. Aggregates per-chunk results into final per-prompt-ID results
5. Generates an Excel report

### Configuration

Test cases and prompts come from the XLSX file configured in `DocSearchConfig.json`:

```json
{
  "doc_assess": {
    "weakness_xlsx_path": "DocAssess/weakness_assessment_prompt.xlsx"
  }
}
```

`_load_settings_constants()` reads this on each call. The workbook contains rows with: `category`, `prompt_id`, `prompt_text`, `expected_values`.

### Key functions

| Function | File | Role |
|---|---|---|
| `assess_doc_v2()` | `DocAssess/assess_v2.py` | Main entry point |
| `chunk_document()` | `DocAssess/assess_v2.py` | Token-based chunking with overlap (uses tiktoken) |
| `evaluate_category_on_chunk()` | `DocAssess/assess_v2.py` | Single LLM call for all prompts in a category on one chunk |
| `aggregate_chunk_results()` | `DocAssess/assess_v2.py` | Merges per-chunk results into final per-prompt result |
| `generate_report()` | `DocAssess/assess_v2.py` | Produces the output report |
| `_load_settings_constants()` | `DocAssess/assess_v2.py` | Loads XLSX test cases + overrides from Django settings |
| `replace_status()` | `DocAssess/output/report.py` | Report post-processing |

### Prompts

Assessment prompts are defined in `DocAssess/Prompts.py` and loaded from `weakness_assessment_prompt.xlsx`. The XLSX is the authoritative source for test cases — add new assessment criteria there without touching code.

### `_build_user_message()`

Constructs the LLM message for a chunk + category combination. Handles JSON extraction from LLM output via `_get_encoder()` (tiktoken-based, singleton) and `_json_from_llm_output()`.

---

## Serializers

| Serializer | File | Used by |
|---|---|---|
| `DocumentAssessSerializers` | `Application/serializer/DocumentAssess.py` | DocAssess V1 |
| `DocumentAssessV2Serializer` | `Application/serializer/DocumentAssessV2.py` | DocAssess V2 |
