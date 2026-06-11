# extra_body — Implementation Record

## What Changed

Added support for `model.extra_body` in `config.yaml` so arbitrary provider-specific
JSON fields can be injected into LLM API requests without waiting for Hermes to expose
them as first-class config keys.

## Files Modified

### 1. `hermes_cli/runtime_provider.py`

Extracts `extra_body` from `model_cfg` at every resolution path (config section,
custom_pool, direct alias, openrouter, etc.) and adds it to the returned runtime dict.

**Lines changed:**
- ~285-288: Added `extra_body = model_cfg.get("extra_body") or {}` and injected into return
- ~501-503: Added extra_body to custom_pool fallback
- ~561: Added extra_body to custom_provider resolution
- ~655-664: Added extra_body before the main custom provider block
- ~762, 802, 823: Added extra_body to various provider return dicts (openai-codex, nous, etc.)

### 2. `run_agent.py`

Accepts `extra_body` as a parameter and passes it through to the transport layer.

**Lines changed:**
- ~1092-1094: Added `extra_body` parameter to `_initialize_agent` signature
- ~1371: Stored `self.extra_body` on the agent instance
- ~9266, ~9361, ~9394: Passed `extra_body_additions=self.extra_body` to transport layer calls

### 3. `agent/transports/chat_completions.py`

Merges `extra_body_additions` into the final API request body. This file was already
updated in a prior commit (commit `c7f0aab9`).

**Merge behavior:**
1. Default provider extra_body (from transport layer)
2. User config extra_body (`config.yaml model.extra_body`) ← **user overrides**
3. Fast mode / CLI request_overrides ← **highest priority**

### 4. `gateway/run.py`

Routes `extra_body` through the messaging gateway so Telegram/Discord/Slack/WhatsApp
sessions also get the params.

**Lines changed:**
- ~707: Added extra_body to runtime_kwargs for direct alias resolution
- ~1881-1895: Added extra_body to gateway route dicts for various resolution paths
- ~10310, ~14932, ~14967: Passed extra_body to agent initialization in gateway turn routing

## Config Example

```yaml
model:
  default: Qwen-3.6
  provider: custom
  base_url: http://localhost:8420/v1
  extra_body:
    temperature: 0.7
    top_p: 0.95
    frequency_penalty: 0.1
```
