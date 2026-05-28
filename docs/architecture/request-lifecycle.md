# Request Lifecycle

Every authenticated API request flows through the same pipeline. Understanding it makes debugging straightforward.

## Pipeline

```
Client                                                    Server
──────                                                    ──────
HTTP request ──────────────────────────────────►  ┌────────────────────────┐
  (API-Key header, JSON body)                     │  Middleware stack      │
                                                  │   LogstashLogging      │  ← logs request + response
                                                  │   CORS                 │
                                                  │   AllowedCIDR          │
                                                  └──────────┬─────────────┘
                                                             │
                                                  ┌──────────▼─────────────┐
                                                  │  URL router            │
                                                  │  DocSearch/urls.py     │
                                                  │  Application/urls.py   │
                                                  └──────────┬─────────────┘
                                                             │
                                                  ┌──────────▼─────────────┐
                                                  │  DRF view              │
                                                  │  IsAdminAPIKeyValid    │  ← terminates auth
                                                  │  IsCompanyAPIKeyValid  │
                                                  └──────────┬─────────────┘
                                                             │ request.data
                                                  ┌──────────▼─────────────┐
                                                  │  Serializer            │
                                                  │  .is_valid()           │  ← validation
                                                  └──────────┬─────────────┘
                                                             │ validated_data
                                                  ┌──────────▼─────────────┐
                                                  │  Feature runner        │
                                                  │  <Feature>/run_query   │  ← all AI logic
                                                  └──────────┬─────────────┘
                                                             │
                                          ┌──────────────────┼──────────────────┐
                                          ▼                  ▼                  ▼
                                      my_llm()         VectorDbManager    models.py
                                  (LLM call)          (Milvus ops)        (ORM)
```

## Step-by-step

### 1. Middleware

`LogstashLoggingMiddleware` (in `Middleware/CustomLoggingMiddleware.py`) logs every request (method, path, body) and response (status, body) to Logstash. It fires on every request regardless of auth outcome.

CORS and allowed-CIDR checks happen here before the request reaches a view.

### 2. URL routing

`DocSearch/urls.py` includes `Application/urls.py` which registers all feature endpoints. Every URL maps to a class-based DRF view in `Application/views/`.

### 3. Authentication

Views declare their auth class:

```python
# Application/views/Chat.py
class DocChatClass(APIView):
    authentication_classes = []
    permission_classes = [IsCompanyAPIKeyValid]
```

`IsCompanyAPIKeyValid` (in `Application/auth.py`) reads the `Api-Key` header, resolves the tenant from it, and either permits or rejects. Admin-only endpoints use `IsAdminAPIKeyValid` instead.

There is no session auth or JWT — every request is stateless and key-authenticated.

### 4. Serializer validation

The view calls the relevant serializer with `request.data`. If `.is_valid()` returns False, the view returns a 400 with `flatten_serializer_errors()` output immediately — the feature runner is never reached.

```python
serializer = DocChatSerializers(data=request.data)
if not serializer.is_valid():
    return Response(flatten_serializer_errors(serializer.errors), status=400)
```

### 5. Feature runner

The view passes `serializer.validated_data` to the relevant feature module (`run_query.py` or equivalent). All LLM calls, Milvus operations, and ORM reads happen inside that module. The view only assembles the final response.

### 6. Response

The feature runner returns structured data; the view wraps it in `Response(data, status=200)`. Errors are surfaced as `Response({"error": "..."}, status=4xx/5xx)`.

## What can go wrong

| Symptom | Likely location |
|---|---|
| 401 / 403 | `Application/auth.py` — invalid or missing `Api-Key` header |
| 400 with field errors | Serializer `.is_valid()` failed — check `Application/serializer/<Feature>.py` |
| 500 from LLM | `LLM/Providers.py::my_llm()` — check Bedrock/OpenAI credentials in `DocSearchConfig.json` |
| 500 from Milvus | `VectorDB/Providers/Milvus/Milvus.py` — check Milvus host/port config |
| Request not logged | `Middleware/CustomLoggingMiddleware.py` — check Logstash connectivity |
