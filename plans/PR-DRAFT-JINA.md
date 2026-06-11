# feat: add Jina as native web search + extract backend

Integrates Jina AI (`s.jina.ai` for search, `r.jina.ai` for extraction) as a
native plugin backend — replacing the previous external Jina MCP tool pattern.

## Why

Jina's search (`s.jina.ai`) and reader (`r.jina.ai`) endpoints provide a unified,
high-quality search+extract stack. Users who already configured Jina via MCP tools
can now switch to the native backend and get the same functionality through the
standard `web_search` / `web_extract` tool calls, with proper backend selection,
fallback, and credential gating.

## What's Changed

### New Files

#### `plugins/web/jina/provider.py`
Core Jina provider implementing `WebSearchProvider` ABC. Uses `httpx` (already a
core dependency) — no extra packages needed.

- **search**: POST to `https://s.jina.ai/` with query body, parses text response
  (split by `---` boundaries) into standard `{"success": True, "data": {"web": [...]}}` shape
- **extract**: GET from `https://r.jina.ai/{url}`, connection-pooled client, returns
  bare list matching legacy `web_extract_tool` post-processing shape
- **sync extract**: No async needed — httpx sync client, dispatcher handles threading
- **error handling**: Per-URL errors for extract, failure fallback for search parse
- **title extraction**: Tries `<title>` tag, markdown heading (`# Title`), first line

#### `plugins/web/jina/__init__.py`
Registers `JinaWebSearchProvider` via `PluginContext.register_web_search_provider()`.

#### `plugins/web/jina/plugin.yaml`
Manifest declaring `provides_web_providers: [jina]`.

### Modified Files

#### `agent/web_search_registry.py`
- Added `"jina"` to `_LEGACY_PREFERENCE` tuple (between `exa` and `searxng`)
- Updated docstring to mention jina in the legacy preference order

#### `agent/web_search_provider.py`
- Updated module docstring to include `jina` in the provider list

#### `tools/web_tools.py` (4 patches)
- `_get_backend()`: added `"jina"` to the `configured` backend check set
- `backend_candidates`: added `("jina", _has_env("JINA_API_KEY"))` between exa and searxng
- `_is_backend_available()`: added `backend == "jina"` → `_has_env("JINA_API_KEY")`
- `_web_requires_env()`: added `"JINA_API_KEY"` to the env var list

#### `pyproject.toml`
- Added `jina = []` extra (httpx is already core, empty extra for backwards compat
  with `hermes tools` and lazy-dep wiring)

#### `tests/plugins/web/test_web_search_provider_plugins.py`
- Updated docstring ("eight" instead of "seven" plugins)
- Added `JINA_API_KEY` to `_clear_web_env` cleanup
- Added `"jina"` to `list_providers()` assertion (plus existing `"xai"`)
- Added `("jina", True, True, False)` to capability parametrize (search + extract, no crawl)
- Added `"jina"` to `name_and_display_name` parametrize
- Added `"jina"` to `setup_schema` parametrize
- Added `test_jina_requires_api_key` (is_available gate)
- Added `test_jina_extract_is_sync` (dispatcher detection)
- Added `test_jina_search_returns_error_dict_when_unconfigured`
- Added `test_jina_extract_returns_per_url_errors_when_unconfigured`

## Env Var

```
JINA_API_KEY=***    # https://jina.ai (required)
```

## Dependency

None — Jina uses `httpx` which is already a core dependency. The `jina` extra in
`pyproject.toml` is empty (`jina = []`) for backwards compat with `hermes tools`.

## Merge Order

1. New plugin files (`plugins/web/jina/`)
2. Registry + tool wiring updates
3. Config schema + pyproject.toml
4. Tests

## Notes

- Jina has a free tier (100 req/day) — the provider doesn't enforce rate limits;
  that's handled by Jina's API
- Search returns text (not JSON) — the `_parse_search_response` function parses
  the `---`-delimited format into standard shape
- Extract is sync (httpx sync client) — the dispatcher's coroutine detection
  handles it the same way
- No `supports_crawl` — Jina doesn't provide a crawl endpoint
