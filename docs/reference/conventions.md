# Conventions

## File naming

| Layer | Convention | Example |
|---|---|---|
| Views | PascalCase class in `Application/views/<Feature>.py` | `DocChatClass`, `AutoTagClass` |
| Serializers | PascalCase `*Serializer(s)` in `Application/serializer/<Feature>.py` | `DocChatSerializers`, `AutoTagSerializer` |
| Feature runners | `run_query.py` in the feature folder | `AutoTag/run_query.py` |
| Prompts | `Prompts/<Feature>.py` or `Prompts/<Feature>/` | `Prompts/AutoTag.py`, `Prompts/RegulatoryGuidelineV2/` |
| Constants | SCREAMING_SNAKE_CASE for module-level constants | `SUMMARISE_PROMPT`, `SQL_GENERATION_PROMPT` |

## View conventions

Every view class must:

1. Inherit from `APIView`
2. Set `authentication_classes = []` (API key auth is permission-class based)
3. Set `permission_classes = [IsCompanyAPIKeyValid]` or `[IsAdminAPIKeyValid]`
4. Return `Response(flatten_serializer_errors(serializer.errors), status=400)` on invalid input
5. Delegate all AI logic to the feature runner

## Serializer conventions

- Use `serializers.Serializer` (not `ModelSerializer`) for request validation
- `validate_<field>()` for field-level business rules
- `validate()` for cross-field rules
- Never call `my_llm()`, `VectorDbManager`, or ORM in a serializer

## Collection naming

Always use `create_collection_name(header, TypeOfSearch.X)`. Never construct a collection name string directly. This ensures multi-tenant isolation.

## Error responses

Always use this shape:

```python
return Response({"error": "<message>", "<context_key>": "<value>"}, status=4xx)
```

For serializer errors:

```python
return Response(flatten_serializer_errors(serializer.errors), status=400)
```

## Import order

```python
# 1. stdlib
import json

# 2. Django / DRF
from django.conf import settings
from rest_framework.views import APIView

# 3. LangChain / third-party
from langchain_core.messages import HumanMessage

# 4. Project: platform layer
from LLM.Providers import my_llm
from VectorDB.VectorDB import VectorDbManager

# 5. Project: feature layer
from Prompts.MyFeature import MY_PROMPT
from Constants.constants import TypeOfSearch
```

## Prompt conventions

- Prompts live in `Prompts/` — not inline in `run_query.py`
- Use `{variable}` placeholders (Python `.format()` style)
- Keep prompts under ~4000 tokens when rendered, accounting for document context
- End structured-output prompts with explicit JSON format instructions

## Versioning features

When a feature gets a major redesign (as with Regulatory Mapping V1 → V2, DocAssess V1 → V2):

- Keep the old module — don't delete or modify it
- Create a new folder (`RegulatoryMappingV2/`, `DocAssess/assess_v2.py`)
- Add new URL under a versioned path (`/regulatory-mapping-v2/`)
- Keep both active until the old version is explicitly deprecated

## TypeOfSearch

Every feature that uses Milvus storage must have its own `TypeOfSearch` value. Adding one:

1. Add the value to the `TypeOfSearch` class in `Constants/constants.py`
2. Use it in `create_collection_name()` calls within your feature runner
3. Document it in the feature's page in this docs site
