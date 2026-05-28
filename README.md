# DocSearch

AI-powered document intelligence platform built with **Django 4.2 + Django REST Framework**. Multi-tenant RAG-based document chat, document assessment, regulatory compliance mapping, and NLP-to-structured-query features. Backed by **Milvus** for vector storage, **LangChain** for LLM orchestration, and **Celery** for async processing.

| Quick links | |
|---|---|
| Design docs (this repo) | `docs/` — served by MkDocs (see [Run the design-docs site](#run-the-design-docs-site)) |
| Django admin | `http://localhost:8000/admin/` |

---

## Table of contents

1. [Prerequisites](#prerequisites)
2. [First-time setup](#first-time-setup)
3. [Run the application](#run-the-application)
4. [Run Celery workers](#run-celery-workers)
5. [Run the design-docs site](#run-the-design-docs-site)
6. [Run with Docker](#run-with-docker)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Tool | Version |
|---|---|
| Python | 3.10+ |
| PostgreSQL | 14+ |
| Redis | 6+ |
| Milvus | 2.x |
| Docker (optional) | 20+ |

External services at runtime (configured via `DocSearchConfig.json`): AWS Bedrock (primary LLM), OpenAI (demo/fallback), HuggingFace (embeddings), Logstash (logging).

---

## First-time setup

```bash
# 1. Clone the repo
git clone https://github.com/Nuva-Org/DocSearch.git
cd DocSearch

# 2. Create a virtual environment
python3 -m venv doc
source doc/bin/activate       # macOS / Linux
# doc\Scripts\Activate.ps1   # Windows

# 3. Install Python dependencies
pip install -r requirements.txt

# 4. Drop in DocSearchConfig.json
#    This file is NOT committed. Get it from a teammate or the secrets pipeline.
#    Place it at the project root (next to manage.py).

# 5. Set up NLTK data (one-time)
python -c "import nltk; nltk.download('punkt'); nltk.download('stopwords')"

# 6. Run migrations
python manage.py migrate
```

> See [docs/architecture/configuration.md](docs/architecture/configuration.md) for the full `DocSearchConfig.json` schema.

---

## Run the application

```bash
# activate the venv first
source doc/bin/activate

# dev server (default port 8000)
python manage.py runserver

# custom port
python manage.py runserver 0.0.0.0:8080
```

Once it's up, API endpoints are available under `/` as registered in `Application/urls.py`.

> Auth scheme is `Api-Key: <key>` header on every endpoint. Admin-only endpoints require an admin-level key. See [docs/reference/conventions.md](docs/reference/conventions.md).

---

## Run Celery workers

```bash
# activate the venv first
source doc/bin/activate

# start a worker
celery -A DocSearch worker -l info

# start beat scheduler
celery -A DocSearch beat -l info
```

Celery is configured in `DocSearch/celery.py`. Redis is used as the broker (configured via `DocSearchConfig.json`).

---

## Run the design-docs site

The platform's design documentation lives in `docs/` and is built with **MkDocs Material**.

```bash
# activate the venv (MkDocs is already installed in the doc venv)
source doc/bin/activate

# install MkDocs if not already present
pip install -r docs/requirements.txt

# live-reload dev server
python -m mkdocs serve
# → http://127.0.0.1:8000

# static build (output in ./site/)
python -m mkdocs build --strict
```

> Run on a different port if 8000 is taken by Django: `python -m mkdocs serve -a 127.0.0.1:8001`

Publishing is manual: trigger the **Build & deploy design docs** workflow under **Actions** in the DocSearchDocs repo. Enable GitHub Pages once under **Settings → Pages → Source = GitHub Actions**.

Start with:

- [docs/index.md](docs/index.md) — landing page
- [docs/feature-guide/contract.md](docs/feature-guide/contract.md) — required reading before adding any new endpoint
- [docs/architecture/overview.md](docs/architecture/overview.md) — platform shape
- [docs/llms.txt](docs/llms.txt) — manifest for feeding the docs to an LLM

---

## Run with Docker

```bash
docker-compose up --build
```

The `docker-compose.yml` at the project root starts Django + Celery. `DocSearchConfig.json` should be mounted or baked in from the secrets pipeline.

The `Dockerfile` base is `python:3.10`. Entrypoint is `script.sh`.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ImportError` on first run | Venv not activated or `requirements.txt` not installed | Activate `doc/`, then `pip install -r requirements.txt` |
| Every API returns `403` | Missing or invalid `Api-Key` header | Add `Api-Key: <key>` to all requests |
| Milvus connection error | Milvus host/port wrong in `DocSearchConfig.json` | Check the `vectordb_provider` config and confirm Milvus is running |
| LLM call fails with auth error | Bedrock/OpenAI credentials wrong or expired | Check `Bedrock_config` / `Default_config` in `DocSearchConfig.json` |
| `NLTK LookupError` | NLTK data not downloaded | `python -c "import nltk; nltk.download('punkt'); nltk.download('stopwords')"` |
| `DocSearchConfig.json` not found | File missing at project root | Get from a teammate — file is gitignored and not committed |
| `mkdocs serve` port collision with Django | Both default to 8000 | `python -m mkdocs serve -a 127.0.0.1:8001` |

For anything not covered here, the design docs are the source of truth:

- [Architecture overview](docs/architecture/overview.md)
- [Request lifecycle](docs/architecture/request-lifecycle.md)
- [Feature creation contract](docs/feature-guide/contract.md)

---

## Useful links

- MkDocs Material: <https://squidfunk.github.io/mkdocs-material/>
- LangChain: <https://python.langchain.com/>
- Milvus: <https://milvus.io/docs>
- AWS Bedrock: <https://aws.amazon.com/bedrock/>
