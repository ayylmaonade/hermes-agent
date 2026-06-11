# `extra_body` — Pass Provider-Specific Params from config.yaml

## What It Does

`extra_body` lets you inject arbitrary JSON fields into the LLM API request body,
from your `config.yaml` model section. These fields are forwarded **verbatim** to the
provider — they bypass all Hermes-level parsing and reach the API raw.

## Why It Exists

Hermes exposes a curated set of model config keys (`default`, `provider`, `base_url`,
`api_key`, `api_mode`, `context_length`, `reasoning_effort`, etc.). But providers
constantly add new parameters that aren't wired into Hermes yet. `extra_body` is the
escape hatch — you can pass any field the provider accepts without waiting for Hermes
to add it.

## Where to Put It

In your `~/.hermes/config.yaml` under the `model:` section:

```yaml
model:
  default: Qwen-3.6
  provider: custom
  base_url: http://localhost:8420/v1
  extra_body:
    temperature: 0.7
    top_p: 0.95
```

## Examples

### OpenRouter Provider Routing

```yaml
model:
  provider: openrouter
  extra_body:
    provider:
      order: [anthropic, google]    # Try these providers first
      sort: throughput               # or "price" | "latency"
    plugins:
      - id: pareto-router
        min_coding_score: 0.5
```

### Gemini Thinking

```yaml
model:
  provider: gemini
  extra_body:
    thinking:
      enabled: true
      budget_tokens: 8192
```

### Temperature & Top-P (not exposed as top-level config)

```yaml
model:
  provider: custom
  base_url: http://localhost:8420/v1
  extra_body:
    temperature: 0.7
    top_p: 0.95
    frequency_penalty: 0.1
```

### Anthropic System Prompt Override

```yaml
model:
  provider: anthropic
  extra_body:
    system: "You are a specialized coding assistant."
```

### OpenRouter API Metadata

```yaml
model:
  provider: openrouter
  extra_body:
    model: anthropic/claude-sonnet-4
```

## How It Merges

The `extra_body` dict from config.yaml is merged into the final API request body
**alongside** any `request_overrides` set by fast mode or the CLI. The merge order is:

1. Default provider extra_body (from transport layer)
2. User config extra_body (`config.yaml model.extra_body`) ← **your overrides**
3. Fast mode / CLI request_overrides

This means your config values win over defaults, and CLI overrides win over config.

## Verification

After setting `extra_body`, you can verify it's reaching the API by checking the
Hermes logs or using a network proxy. The fields appear in the raw request body
sent to the provider endpoint.

## Notes

- Keys that conflict with top-level model config keys (e.g. `model`, `provider`)
  may be overwritten by Hermes' own request construction. Stick to fields that
  are only in the request body, not the URL or headers.
- For OpenRouter-specific features, use the dedicated `openrouter:` config section
  (`response_cache`, `provider_routing`) — these are better maintained than raw
  `extra_body` overrides.
- Not all providers support all fields. Check your provider's API docs for valid
  request body parameters.
