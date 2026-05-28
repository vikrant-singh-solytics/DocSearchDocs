# Configuration

All runtime configuration is loaded from a single JSON file: **`DocSearchConfig.json`** at the project root. Parsed once at start in `DocSearch/settings.py` and exposed as module-level variables.

## File location

| Environment | Source |
|---|---|
| Local dev | `DocSearchConfig.json` at the project root (gitignored). Copy from a teammate or the secrets pipeline. |
| Docker / production | Mounted via volume from the secrets pipeline. |

## Top-level sections

| Section | Purpose |
|---|---|
| `Default_config` | Default provider selection: `llm_provider`, `embeddings_provider`, `vectordb_provider`, `api_key`. |
| `Demo_config` | Demo/testing override — same shape as `Default_config`, used when `Demo: true`. |
| `Bedrock_config` | AWS Bedrock production config: `model_id`, `region_name`, `bedrock_api_key`. |
| `Demo` | Boolean — `true` uses `Demo_config`, `false` uses `Default_config`. |
| `bedrock` | Boolean — `true` uses `Bedrock_config` for LLM instead of `Default_config`. |
| `Logstash` | `logstash_host` + `logstash_port` for request/response logging. |
| `doc_assess` | `weakness_xlsx_path` — path to the DocAssess test-case workbook. |

## How it's exposed in Python

```python
# DocSearch/settings.py loads DocSearchConfig.json into module-level vars
from django.conf import settings

config = settings.DOC_SEARCH_CONFIG
llm_provider   = config["Default_config"]["llm_provider"]
logstash_host  = config["Logstash"]["logstash_host"]
```

Import from `django.conf.settings`. Don't read `DocSearchConfig.json` again at runtime.

## Provider selection

```
bedrock == true  →  Bedrock_config
Demo == true     →  Demo_config
otherwise        →  Default_config
```

`my_llm()` and `my_embedding()` implement this logic automatically.

## Adding a new setting

1. Add the key to `DocSearchConfig.json`.
2. In `DocSearch/settings.py`, parse it into a module-level constant.
3. Import that constant where needed via `django.conf.settings`.
4. Document the key in this page.
5. **Never** hardcode secrets or environment-specific values in code.
