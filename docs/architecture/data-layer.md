# Data Layer — Models

DocSearch's persistent state lives in two places: **PostgreSQL** (Django ORM for structured records) and **Milvus** (vector storage for embeddings). This page covers the Django side. For Milvus, see [VectorDB](vectordb.md).

## Models (`Application/models.py`)

Four Django models form the core of the platform:

### `Tenant`

Represents an organisation/customer. Every document, session, and API key belongs to a tenant.

```python
class Tenant(models.Model):
    # tenant identifier — appears in collection names and API key lookups
```

### `Document`

A document uploaded to the platform. Tracks metadata and links to the tenant.

```python
class Document(models.Model):
    documentid   # UUID7 primary key
    # file metadata, tenant FK, upload timestamp, etc.
```

### `Session`

A chat session grouping a sequence of `ChatBox` turns. Scoped to a tenant and optionally to a document.

```python
class Session(models.Model):
    # tenant FK, document FK, creation metadata
```

### `ChatBox`

One turn in a chat session: a user message + the AI response.

```python
class ChatBox(models.Model):
    session       # FK → Session
    result_content  # the AI answer stored for context window management
    # user message, timestamps, etc.
```

## ORM access rule

**`Application/models.py` is the only file that defines models. Views and feature runners access the ORM via Django's standard queryset API — but only for the four models above.** LLM calls, Milvus queries, and any AI processing must live in the feature runner (`run_query.py`), not in views or serializers.

## Migrations

Migrations live in `Application/migrations/`. Run them with:

```bash
python manage.py migrate
```

The migration history (0001–0007 as of this writing) tracks structural changes like UUID field changes and column removals. Always generate a new migration for model changes:

```bash
python manage.py makemigrations
```

## Data flow for a typical request

```
POST /chat/
  │
  ▼ serializer validates
  ▼ view calls Chat/run_query.py
      │
      ├─► Session.objects.get(...)         # load session
      ├─► ChatBox.objects.filter(...)      # load prior turns
      │
      ├─► VectorDbManager.search(...)      # fetch relevant chunks from Milvus
      ├─► my_llm()(prompt)                 # LLM call
      │
      └─► ChatBox.objects.create(...)      # save the new turn
```

## Key utilities

**`Application/Utilities/Document.py`** — `extract_text_from_file()` handles PDF (via `pdfplumber`), DOCX, Excel, and plain text. Called during document ingestion before chunking and embedding.

**`Application/Utilities/RAGCommon.py`** — `create_collection_name()` builds a deterministic Milvus collection name from a header/API key and a `TypeOfSearch` value. This is how multi-tenancy is enforced at the vector layer. `create_response()` standardises the response envelope.

**`Constants/constants.py`** — `TypeOfSearch` is the platform's central enum, used as a namespace discriminator for Milvus collections across all features (chat, regulatory mapping, SQL schema, etc.). Also houses `flatten_serializer_errors()` for consistent 400 responses.
