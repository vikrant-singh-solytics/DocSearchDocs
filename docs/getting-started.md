# Getting Started

## Prerequisites

| Tool | Version |
|---|---|
| Python | 3.10+ |
| PostgreSQL | 14+ |
| Redis | 6+ |
| Milvus | 2.x |
| Docker (optional) | 20+ |

External services at runtime (configured via `DocSearchConfig.json`): PostgreSQL, Redis, Milvus, AWS Bedrock (primary LLM), OpenAI (demo/fallback), HuggingFace (embeddings), Logstash (logging).

## Configuration

All runtime configuration lives in **`DocSearchConfig.json`** at the project root. It is **not** committed. Get it from the deployment/secrets pipeline or copy from a teammate. See [Configuration](architecture/configuration.md) for the full schema.

## Local development

```bash
# create a virtual environment (Python 3.10+)
python3 -m venv venv
source venv/bin/activate          # macOS / Linux
# venv\Scripts\Activate.ps1       # Windows

# install dependencies
pip install -r requirements.txt

# run migrations
python manage.py migrate

# set up NLTK data (required once)
python -c "import nltk; nltk.download('punkt'); nltk.download('stopwords')"

# run dev server
python manage.py runserver
```

Django admin is at `/admin/`. Swagger UI is not bundled by default — use the `Application/urls.py` route list as the API reference.

## Docker

```bash
docker-compose up --build
```

The `docker-compose.yml` at the project root starts Django + Celery together. Environment variables are passed via `.env` or the compose file directly.

## Celery

```bash
# start a worker
celery -A DocSearch worker -l info

# start beat scheduler
celery -A DocSearch beat -l info
```

Celery is configured in `DocSearch/celery.py`. Tasks are dispatched for async operations (e.g. document processing jobs).

## Running tests

```bash
pytest
# or
python manage.py test
```

## Building these docs locally

```bash
pip install -r docs/requirements.txt
mkdocs serve     # http://127.0.0.1:8000/DocSearch/
mkdocs build     # static site → site/
```

GitHub Pages publishes from the `gh-pages` branch via `.github/workflows/docs.yml` on every manual trigger.
