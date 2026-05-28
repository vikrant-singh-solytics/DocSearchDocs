# The Feature Creation Contract

This is the contract every new endpoint in DocSearch must follow. A PR that breaks it is not mergeable.

## The flow, in one diagram

```
1. Application/urls.py                       (register the route)
        │
        ▼
2. Application/views/<Feature>.py            (class-based view; one method per verb)
        │
        ▼
3. Application/serializer/<Feature>.py       (validate the request)
        │
        ▼
4. <Feature>/run_query.py                    (all AI + business logic)
        │
        ├─► LLM/Providers.py :: my_llm()     (LLM calls)
        ├─► VectorDB/VectorDB.py             (Milvus operations)
        ├─► Embeddings/Providers.py          (embeddings)
        └─► Application/models.py            (ORM — session / document / chatbox)
```

## The four steps

### 1. URL — `Application/urls.py`

Add one `path()` entry pointing at a view from `Application/views/<Feature>.py`.

```python
# Application/urls.py
from Application.views.MyFeature import MyFeatureView

urlpatterns = [
    ...
    path("my-feature/", MyFeatureView.as_view(), name="my-feature"),
]
```

### 2. View — `Application/views/<Feature>.py`

Class-based. One method per HTTP verb. The view's only jobs are: auth, call the serializer, call the feature runner, return the response.

```python
# Application/views/MyFeature.py
from rest_framework.views import APIView
from rest_framework.response import Response
from Application.auth import IsCompanyAPIKeyValid
from Application.serializer.MyFeature import MyFeatureSerializer
from Constants.constants import flatten_serializer_errors
from MyFeature.run_query import run_my_feature

class MyFeatureView(APIView):
    authentication_classes = []
    permission_classes = [IsCompanyAPIKeyValid]

    def post(self, request):
        serializer = MyFeatureSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(flatten_serializer_errors(serializer.errors), status=400)

        result = run_my_feature(**serializer.validated_data)
        return Response(result, status=200)
```

**Rules:**
- Always use `IsCompanyAPIKeyValid` (or `IsAdminAPIKeyValid` for admin-only endpoints).
- Never put AI logic, ORM queries, or LLM calls in the view.
- Always call `flatten_serializer_errors()` for 400 responses.

### 3. Serializer — `Application/serializer/<Feature>.py`

Validates the incoming request body. Add custom `validate_<field>()` methods for cross-field validation.

```python
# Application/serializer/MyFeature.py
from rest_framework import serializers

class MyFeatureSerializer(serializers.Serializer):
    document_id = serializers.UUIDField()
    query       = serializers.CharField(max_length=2000)
    header      = serializers.CharField()   # tenant API key

    def validate_query(self, value):
        if len(value.strip()) < 3:
            raise serializers.ValidationError("Query too short.")
        return value
```

### 4. Feature runner — `<Feature>/run_query.py`

All AI logic lives here: LLM calls, Milvus queries, embeddings, ORM reads. Returns a plain Python dict or list — no `Response` objects.

```python
# MyFeature/run_query.py
from LLM.Providers import my_llm
from VectorDB.VectorDB import VectorDbManager
from Application.Utilities.RAGCommon import create_collection_name
from Constants.constants import TypeOfSearch
from Prompts.MyFeature import MY_FEATURE_PROMPT

def run_my_feature(document_id, query, header):
    collection = create_collection_name(header, TypeOfSearch.MY_FEATURE)
    vdb = VectorDbManager()

    # retrieve context
    docs = vdb.search(collection, query, top_k=5)
    context = "\n".join(d.page_content for d in docs)

    # LLM call
    llm = my_llm()
    answer = llm.invoke(MY_FEATURE_PROMPT.format(context=context, query=query))

    return {"answer": answer.content}
```

**Rules:**
- Never return a `Response` object — return plain data.
- Use `my_llm()` — never construct an LLM client directly.
- Use `VectorDbManager` — never import `pymilvus` directly.
- Add a `TypeOfSearch` value for the new feature (in `Constants/constants.py`).
- Add a prompt in `Prompts/MyFeature.py`.

## What NOT to do

| Don't | Do instead |
|---|---|
| `LLM call in a view` | Call `run_query.py` from the view |
| `import pymilvus in run_query.py` | Use `VectorDbManager` |
| `import anthropic / openai directly` | Use `my_llm()` |
| `Inline prompt strings in run_query.py` | Put prompts in `Prompts/MyFeature.py` |
| `ORM query in a serializer` | ORM calls belong in `run_query.py` only |
| `Hardcode Milvus collection name` | Use `create_collection_name()` |
