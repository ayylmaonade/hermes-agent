# Hermes API Parameter Patch — Documentation

This directory collects all documentation for the `extra_body` patch, which allows
passing provider-specific parameters (temperature, top_p, OpenRouter routing, etc.)
directly from `config.yaml` into the LLM API request body.

## Contents

| File | Description |
|------|-------------|
| [EXTRA_BODY.md](EXTRA_BODY.md) | Full usage guide — config examples, merge behavior, provider-specific features |
| [IMPLEMENTATION.md](IMPLEMENTATION.md) | Patch record — file-by-file implementation details and diffs |

## Implementation Files

The patch touches these files in the Hermes codebase:

| File | What It Does |
|------|--------------|
| `hermes_cli/runtime_provider.py` | Resolves `extra_body` from config at runtime |
| `run_agent.py` | Passes `extra_body` into the agent loop |
| `agent/transports/chat_completions.py` | Merges `extra_body` into the final API request body |
| `gateway/run.py` | Routes `extra_body` through the messaging gateway |

## Quick Config Example

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
