# Regulatory Mapping

Two versions exist. V1 is a direct RAG-based evaluation; V2 is a sophisticated decompose-evaluate-summarise pipeline that handles multiple entities and model types.

---

## Regulatory Mapping V1

**Endpoint:** `POST /regulatory-mapping/`  
**View:** `Application/views/RegMapping.py::RegulatoryGuideLineView`  
**Runner:** `RegulatoryMapping/run_query.py`  
**Auth:** `IsCompanyAPIKeyValid`

### What it does

Evaluates model documents against a regulatory policy/guideline using RAG. Returns structured pass/partial/fail findings.

### Key functions

| Function | Role |
|---|---|
| `regulatory_mapping_query()` | Core evaluation — calls `my_llm()` per guideline chunk |
| `regulatory_mapping_with_consolidation()` | Enhanced version — consolidates multi-chunk results |
| `consolidate_regulatory_mapping_results()` | Merges findings from multiple chunks into one cohesive result |

`RegulatoryMapping/Utilities.py::chunk_text()` splits the regulatory policy text before evaluation.

---

## Regulatory Mapping V2

**Endpoints:**  
- `POST /regulatory-mapping-v2/` → `RegulatoryGuideLineViewV2`  
- `POST /regulatory-mapping-v2/decompose/` → `RegulatoryGuidelineDecomposeView`

**Runner:** `RegulatoryMappingV2/run_query.py`  
**Auth:** `IsCompanyAPIKeyValid`

### What it does

A full compliance assessment pipeline across multiple model entities:

```
1. decompose_policy()         — extract all testable requirements from the policy doc
2. identify_model_type()      — determine the type of model being assessed
3. filter_applicable_tests()  — select tests relevant to this model type
4. get_augmentation_checks()  — add baseline checks per model type
5. evaluate_all_tests()       — run each test against each entity's Milvus context
6. deduplicate_findings()     — collapse duplicate PARTIAL/MISSING findings
7. aggregate_entity_result()  — compute final verdict per entity
8. compute_final_verdict()    — overall pass/partial/fail across all entities
9. generate_executive_summary() — single-paragraph summary
```

### Key files

| File | Role |
|---|---|
| `RegulatoryMappingV2/run_query.py` | Full pipeline |
| `RegulatoryMappingV2/augmentation_checks.py::get_augmentation_checks()` | Baseline checks per model type |
| `Prompts/RegulatoryGuidelineV2/decompose_policy.py` | Decomposition prompt |
| `Prompts/RegulatoryGuidelineV2/evaluate_test.py` | Per-test evaluation prompt |
| `Prompts/RegulatoryGuidelineV2/generate_summary.py` | Executive summary prompt |
| `Prompts/RegulatoryGuidelineV2/identify_model_type.py` | Model type identification prompt |

### `build_entity_chain()`

Creates a LangChain `RetrievalQA` chain scoped to one entity's Milvus collection. Used in `evaluate_all_tests()` to give each entity its own retrieval context during parallel evaluation.

### Team context

`_build_team_context()` formats entity team metadata (roles, responsibilities) into a readable string injected into evaluation prompts. This allows the LLM to consider organisational accountability when making compliance judgements.

### Consolidation

`deduplicate_findings()` collapses PARTIAL/MISSING findings that share a root cause. ABSENCE findings (where the requirement is entirely missing) are never collapsed — they must appear individually in the final report.

---

## Serializers

| Serializer | File |
|---|---|
| `RegulatoryGuidelineSerializers` | `Application/serializer/RegulatoryGuideline.py` |
