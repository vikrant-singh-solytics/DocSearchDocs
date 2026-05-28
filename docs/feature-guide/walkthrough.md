# Worked Example — Adding a Document Summarisation Endpoint

This walkthrough adds a real endpoint: `POST /summarise/` that takes a document ID and returns an AI-generated summary.

## Step 1: Add a `TypeOfSearch` value

```python
# Constants/constants.py
class TypeOfSearch:
    DOCUMENT_SEARCH = "document_search"
    REGULATORY_MAPPING = "regulatory_mapping"
    # ... existing values ...
    SUMMARISE = "summarise"           # ← add this
```

## Step 2: Add the prompt

```python
# Prompts/Summarise.py
SUMMARISE_PROMPT = """You are a precise document summariser.

Context from the document:
{context}

Produce a concise summary (3-5 sentences) covering the main purpose, key findings, and any critical constraints.
Return only the summary text."""
```

## Step 3: Write the feature runner

```python
# Summarise/run_query.py
from LLM.Providers import my_llm
from VectorDB.VectorDB import VectorDbManager
from Application.Utilities.RAGCommon import create_collection_name
from Constants.constants import TypeOfSearch
from Prompts.Summarise import SUMMARISE_PROMPT

def summarise_document(document_id: str, header: str) -> dict:
    collection = create_collection_name(header, TypeOfSearch.SUMMARISE)
    vdb = VectorDbManager()

    # pull up to 10 chunks — summarisation needs more context than Q&A
    docs = vdb.search(collection, query="", top_k=10, filter=f'file_id == "{document_id}"')
    if not docs:
        return {"error": "No content found for this document."}

    context = "\n\n".join(d.page_content for d in docs)
    llm = my_llm()
    response = llm.invoke(SUMMARISE_PROMPT.format(context=context))

    return {"summary": response.content, "document_id": document_id}
```

## Step 4: Write the serializer

```python
# Application/serializer/Summarise.py
from rest_framework import serializers

class SummariseSerializer(serializers.Serializer):
    document_id = serializers.CharField()
    header      = serializers.CharField()
```

## Step 5: Write the view

```python
# Application/views/Summarise.py
from rest_framework.views import APIView
from rest_framework.response import Response
from Application.auth import IsCompanyAPIKeyValid
from Application.serializer.Summarise import SummariseSerializer
from Constants.constants import flatten_serializer_errors
from Summarise.run_query import summarise_document

class SummariseView(APIView):
    authentication_classes = []
    permission_classes = [IsCompanyAPIKeyValid]

    def post(self, request):
        serializer = SummariseSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(flatten_serializer_errors(serializer.errors), status=400)
        result = summarise_document(**serializer.validated_data)
        return Response(result, status=200)
```

## Step 6: Register the URL

```python
# Application/urls.py  — add one line
from Application.views.Summarise import SummariseView

urlpatterns = [
    ...
    path("summarise/", SummariseView.as_view(), name="summarise"),
]
```

## Step 7: Add to `FeatureModels` (if using Bedrock)

```python
# LLM/FeatureModels.py
class FeatureModels:
    ...
    SUMMARISE = "us.anthropic.claude-haiku-4-5"   # cheaper model for summarisation
```

And map it in `detect_from_stack()`.

## Checklist before opening a PR

- [ ] `TypeOfSearch` value added
- [ ] Prompt in `Prompts/`
- [ ] `run_query.py` uses `my_llm()` and `VectorDbManager`
- [ ] Serializer validates all inputs
- [ ] View has `permission_classes = [IsCompanyAPIKeyValid]`
- [ ] URL registered in `Application/urls.py`
- [ ] No `pymilvus`, `anthropic`, or `openai` imports in feature code
- [ ] No prompt strings inline in `run_query.py`
