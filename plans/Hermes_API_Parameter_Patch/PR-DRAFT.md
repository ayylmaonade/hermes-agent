# feat: add 'model.extra_body' to allow passing of custom API params for main model (incl. OpenRouter routing)

aux models already have `extra_body` support (vision, web_extract, compression, etc.)
so this extends the same concept to the main model — lets users inject arbitrary API
params from config.yaml without waiting for Hermes to expose each provider knob.

## Why

Hermes exposes a curated set of model keys (provider, default, base_url,
reasoning_effort). Providers constantly add new parameters not yet wired into
Hermes. `model.extra_body` is the escape hatch — pass any field the provider
accepts without waiting.

## What's Changed

### run_agent.py
- AIAgent takes `extra_body` kwarg, stores it as `self.extra_body`
- Passes it as `extra_body_additions` to all 3 transport call sites (ChatCompletions,
  provider profile, legacy flag path)

### hermes_cli/runtime_provider.py
- All runtime resolution functions read `model_cfg.get("extra_body")` and include
  it in the returned dicts
- Covers: pool entry, named custom, openrouter, azure foundry, explicit runtime

### gateway/run.py
- Route construction dicts carry `extra_body` so per-session agents inherit it
- 6 locations updated: kwargs dicts and turn routes

### cli-config.yaml.example
- Doc comment showing how to use it (temperature, top_p, provider routing, thinking)

## Merge Order

1. Default provider extras (transport built-ins)
2. User config extra_body ← your overrides win
3. CLI/fetch request_overrides

## Example

```yaml
model:
  provider: openrouter
  extra_body:
    provider:
      order: [anthropic, google]
      sort: throughput
```

```yaml
model:
  provider: custom
  base_url: http://localhost:8420/v1
  extra_body:
    temperature: 0.7
    top_p: 0.95
    frequency_penalty: 0.1
```

## Notes

- No validation or sanitization — fields pass verbatim to provider. User controls
  their own API requests. Keys conflicting with top-level config keys (model,
  provider) may be overwritten by Hermes' own request construction.
- For OpenRouter-specific features, use the dedicated `openrouter:` config section
  (response_cache, provider_routing) — those are better maintained.