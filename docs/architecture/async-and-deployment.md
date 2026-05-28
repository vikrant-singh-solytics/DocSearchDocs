# Async Processing & Deployment

## Celery

- **Broker**: Redis (configured via `DocSearchConfig.json`).
- **App**: `DocSearch.celery.app` (`DocSearch/celery.py`).
- Celery is used for async document processing tasks where long-running AI operations should not block the HTTP response.

### Defining a task

```python
# DocSearch/celery.py registers the app
from DocSearch.celery import app

@app.task
def process_document_async(document_id, tenant_key):
    # 1. load document from DB
    # 2. extract text
    # 3. chunk + embed + ingest into Milvus
    pass
```

### Dispatching

```python
process_document_async.delay(document_id, tenant_key)
```

## Deployment

`script.sh` is the Docker entrypoint. The `docker-compose.yml` at the project root defines the service topology:

| Container | What runs |
|---|---|
| `web` | `python manage.py runserver` (dev) or Gunicorn (prod) |
| `celery` | `celery -A DocSearch worker -l info` |

### Runtime config

`DocSearchConfig.json` is mounted at the project root and parsed at Django start in `DocSearch/settings.py`. Changes require a restart.

### ASGI / WSGI

- `DocSearch/asgi.py` — ASGI entrypoint (for async-capable servers like Daphne/Uvicorn)
- `DocSearch/wsgi.py` — WSGI entrypoint (for Gunicorn)

The `Dockerfile` at the project root builds the image from `python:3.10`.

## Logging

All requests are logged by `Middleware/CustomLoggingMiddleware.py` (`LogstashLoggingMiddleware`):

- `process_request()` — logs method, path, and body
- `process_response()` — logs status and response body
- `process_exception()` — logs unhandled exceptions with full traceback

Logstash host and port come from `DocSearchConfig.json → Logstash`:

```json
{
  "Logstash": {
    "logstash_host": "docsearch-logstash.solytics.us",
    "logstash_port": 10960
  }
}
```

LLM token usage is logged by `TokenUsageLogger` in `LLM/Providers.py` — each call records input/output tokens and the calling feature.

## Monitoring

| Signal | Location |
|---|---|
| Request/response logs | Logstash (via `LogstashLoggingMiddleware`) |
| LLM token usage | `TokenUsageLogger` callback (per request) |
| Errors | Logged by `process_exception()`, also surfaced as 500 responses |

## Failure handling

- **Validation failures** are caught by DRF serializers and returned as 400 with `flatten_serializer_errors()` output.
- **LLM call failures** bubble up from `my_llm()` — wrap calls in try/except in the feature runner for graceful degradation.
- **Milvus failures** surface from `VectorDbManager` — connection errors should be caught at the view level and returned as 503.
- **Celery task failures** are retried per `@app.task(autoretry_for=..., max_retries=...)` decorations.
