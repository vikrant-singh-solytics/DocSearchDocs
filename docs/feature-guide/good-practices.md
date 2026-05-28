# Good Practices

## LLM calls

**Use `my_llm()` — always.**

```python
# ✓ correct
from LLM.Providers import my_llm
llm = my_llm()
response = llm.invoke(prompt)

# ✗ wrong — bypasses token logging and provider switching
import anthropic
client = anthropic.Anthropic()
```

**Never put the LLM call in a view or serializer.** It belongs in `run_query.py`.

**Keep prompts in `Prompts/`.** A feature runner that constructs long prompt strings inline is a code smell — the prompt will need tweaking, and inline strings are harder to version and review.

**Use `FeatureModels` for Bedrock.** Match the right model size to the task — don't default everything to the most expensive model.

## Vector operations

**Use `create_collection_name()` for every collection name.** Never hardcode or construct a collection name in feature code. The function enforces the `{tenant_key}_{TypeOfSearch}` naming convention that keeps collections isolated.

**Use `TypeOfSearch` to namespace collections.** Every feature gets its own `TypeOfSearch` value. Two features that share a collection will corrupt each other's data.

**Filter by `file_id` where possible.** Broad searches without a file scope return chunks from all documents for that tenant. For single-document features (DocAssess, Summarise) always pass a metadata filter.

## Serializer validation

**Validate all inputs at the serializer layer.** Never validate in the view or in `run_query.py`. The serializer is the trust boundary.

**Use `flatten_serializer_errors()` for every 400 response.**

```python
if not serializer.is_valid():
    return Response(flatten_serializer_errors(serializer.errors), status=400)
```

**Add `validate_<field>()` for business rules.** Length checks, allowed value sets, and cross-field constraints belong in the serializer, not the runner.

## Auth

**Every endpoint must declare an explicit `permission_classes`.**

```python
permission_classes = [IsCompanyAPIKeyValid]   # for tenant APIs
permission_classes = [IsAdminAPIKeyValid]     # for admin-only APIs
```

Never leave `permission_classes = []` on a production endpoint.

## Performance

**Limit `top_k` to what the LLM context window can absorb.** A Milvus search with `top_k=50` stuffed into a single prompt is wasteful. Start with 5–10 and tune.

**Use token budget suppression for long chat histories.** The `_suppress_segment()` + `_token_estimate()` pattern in `Application/views/Chat.py` is the reference implementation — reuse it when building features that maintain conversation history.

**Avoid loading large XLSX files on every request.** `DocAssess/assess_v2.py::_load_settings_constants()` is called once per `assess_doc_v2()` invocation. For hot paths, load the workbook into a module-level cache at startup.

## Error handling

**Return structured errors, not bare strings.**

```python
# ✓
return Response({"error": "Document not found.", "document_id": doc_id}, status=404)

# ✗
return Response("Document not found", status=404)
```

**Catch provider-specific exceptions in `run_query.py`**, not in the view. The view should only see a clean dict or a raised exception; it should not need to know whether the failure came from Milvus or Bedrock.

## Code organisation

**One feature per folder.** `AutoTag/`, `SuggestFields/`, `TextToSQL/` — each is self-contained with its own `run_query.py`, a `Prompts/` entry, and a serializer. Do not add a new feature by appending to an existing module.

**Keep `run_query.py` thin at the top level.** Long private helpers (`_chunk_document`, `_build_user_message`, `_parse_llm_json`) are fine — keep the public entry-point function readable in one screen.
