# Review Checklist

Use this checklist when reviewing any PR that adds or modifies a feature.

## Architecture

- [ ] The view does not contain LLM calls, Milvus queries, or ORM reads beyond what is in `Application/models.py`
- [ ] The view has an explicit `permission_classes` — never `[]`
- [ ] The serializer validates all inputs; business-rule checks are in `validate_<field>()` methods
- [ ] 400 responses use `flatten_serializer_errors()`
- [ ] Feature logic lives in `<Feature>/run_query.py`, not in the view

## LLM

- [ ] `my_llm()` is used — no direct `anthropic`, `openai`, or `boto3` imports in feature code
- [ ] Prompts live in `Prompts/<Feature>.py` — no long inline strings in `run_query.py`
- [ ] `FeatureModels` entry added if using Bedrock (correct model size for the task)
- [ ] Token usage is automatically logged (this is free if you use `my_llm()`)

## VectorDB

- [ ] `VectorDbManager` is used — no direct `pymilvus` imports in feature code
- [ ] Collection name is built with `create_collection_name(header, TypeOfSearch.X)`
- [ ] A `TypeOfSearch` value exists for this feature
- [ ] Metadata filters are applied where operations are per-document or per-table

## URL & routing

- [ ] New URL registered in `Application/urls.py`
- [ ] URL name is kebab-case and unique

## Error handling

- [ ] LLM and Milvus errors are caught in `run_query.py` and re-raised or returned as structured dicts
- [ ] Empty-result cases are handled (e.g. no chunks found in Milvus)
- [ ] The view always returns a `Response` — no unhandled exceptions escape to Django's 500 handler

## General

- [ ] No secrets or API keys committed
- [ ] No hardcoded Milvus host, collection name, or model ID
- [ ] Prompt templates do not leak sensitive data (they may be logged)
- [ ] `DocSearchConfig.json` schema documented if a new key was added
