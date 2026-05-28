# LLM & Embeddings

DocSearch's AI capabilities are abstracted behind two factory functions. No feature module ever constructs an LLM or embedding client directly.

## LLM factory — `my_llm()`

**File:** `LLM/Providers.py`

```python
from LLM.Providers import my_llm

llm = my_llm()          # returns a LangChain BaseChatModel
response = llm.invoke(prompt)
```

`my_llm()` reads `DocSearchConfig.json` and returns the configured provider:

| `llm_provider` | Backend |
|---|---|
| `"Anthropic"` | `langchain-anthropic` (Claude) |
| `"OpenAI"` | `langchain-openai` (GPT) |
| `"Bedrock"` | `langchain-aws` (AWS Bedrock — default in production) |

### Token usage logging

Every LLM call is wrapped by `TokenUsageLogger` (a LangChain `BaseCallbackHandler`). It records input/output tokens per call and tags them with the calling feature via `FeatureModels.detect_from_stack()`.

### Feature → Bedrock model registry (`LLM/FeatureModels.py`)

`FeatureModels` maps each platform feature to a specific Bedrock model ID. This lets the platform use different model sizes for different tasks without scattering model IDs across feature code.

```python
# LLM/FeatureModels.py
class FeatureModels:
    CHAT       = "us.anthropic.claude-sonnet-4-6"
    AUTOTAG    = "us.anthropic.claude-haiku-4-5"
    DOC_ASSESS = "us.anthropic.claude-sonnet-4-6"
    # ...
```

`detect_from_stack()` inspects the Python call stack to identify which feature is calling `my_llm()` and picks the right model. Adding a new feature: add its key to `FeatureModels` and map it in `detect_from_stack()`.

`calculate_cost()` computes estimated dollar cost from token counts using Bedrock pricing.

## Embedding factory — `my_embedding()`

**File:** `Embeddings/Providers.py`

```python
from Embeddings.Providers import my_embedding

embeddings = my_embedding()     # returns a LangChain Embeddings instance
vector = embeddings.embed_query(text)
```

Reads `embeddings_provider` from `DocSearchConfig.json`:

| `embeddings_provider` | Backend |
|---|---|
| `"HuggingFace"` | `sentence-transformers` (local, default) |
| `"OpenAI"` | `langchain-openai` embeddings |

## Configuration

Both factories read from `DocSearchConfig.json`. In production the `Bedrock_config` block is used:

```json
{
  "Bedrock_config": {
    "model_id": "us.anthropic.claude-sonnet-4-6",
    "region_name": "us-east-1",
    "bedrock_api_key": "..."
  },
  "Default_config": {
    "llm_provider": "Anthropic",
    "embeddings_provider": "HuggingFace",
    "api_key": "..."
  }
}
```

See [Configuration](configuration.md) for the full schema.

## Prompt templates

All prompt strings live in `Prompts/`. Each feature has its own module:

| Module | Feature |
|---|---|
| `Prompts/basic.py`, `Prompts/prompts.py` | Shared / base prompts |
| `Prompts/AutoTag.py` | AutoTag |
| `Prompts/RegulatoryGuideline.py` | Regulatory Mapping V1 |
| `Prompts/RegulatoryGuidelineV2/` | Regulatory Mapping V2 (decompose / evaluate / summarise) |
| `Prompts/SuggestFields.py` | SuggestFields |
| `Prompts/TextToSql.py` | Text-to-SQL |
| `Prompts/TextToFilter.py` | Text-to-Filter |
| `Prompts/TextToWorkflow.py` | Text-to-Workflow |
| `Prompts/vault_model.py` | Model Chat |

**Rule:** prompt strings belong in `Prompts/`. Feature runners import from there. Never inline a multi-line prompt string in a `run_query.py`.

## Adding a new LLM-powered feature

1. Add a `Prompts/MyFeature.py` with the prompt template.
2. Add a `FeatureModels.MY_FEATURE` constant and a branch in `detect_from_stack()`.
3. Call `my_llm()` in your `run_query.py` — token logging happens automatically.
4. Never import `boto3`, `anthropic`, or `openai` directly in feature code.
