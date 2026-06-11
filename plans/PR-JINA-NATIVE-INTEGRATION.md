# feat: add Jina web search + extract provider plugin

## Summary

Adds Jina AI as a pluggable web search and content extraction backend for Hermes Agent. Jina provides two REST endpoints:
- **Search** (`https://s.jina.ai/`) — web search with results (text/plain response)
- **Reader** (`https://r.jina.ai/`) — URL extraction with LLM-cleaned content (text/plain response)

This is the 8th and final web-provider plugin. All providers now live as plugins under `plugins/web/<name>/` (established by PR #25182, which moved the original 7 providers from in-tree hardcoded modules into the pluggable architecture).

**No Python SDK needed** — Jina is a pure REST API, so the only dependency is `httpx` which is already a core dep.

## Files

### New files
| File | Purpose |
|------|---------|
| `plugins/web/jina/provider.py` | Provider implementation (`JinaWebSearchProvider`) |
| `plugins/web/jina/plugin.yaml` | Plugin manifest (`kind: backend`, `provides_web_providers: [jina]`) |
| `plugins/web/jina/__init__.py` | Plugin entry point — registers provider via `ctx.register_web_search_provider()` |

### Modified files
| File | Changes |
|------|---------|
| `agent/web_search_registry.py` | Added `"jina"` to `_LEGACY_PREFERENCE` tuple (after exa, before searxng) |
| `agent/web_search_provider.py` | Added `jina` to provider list in docstring; **fixed extract return shape** (was `{"success": True, "data": [...]}`, corrected to bare list `[...]` to match actual implementation) |
| `tools/web_tools.py` | Added `"jina"` to 4 locations: `_get_backend()` config accept list, `backend_candidates` auto-detect, `_is_backend_available()` switch, `_web_requires_env()` env vars list; updated module docstring and comment counts |
| `hermes_cli/tools_config.py` | Updated comment to reference 8 providers instead of 7 |
| `pyproject.toml` | Added `jina = []` optional dependency (httpx is already a core dep) |
| `tests/plugins/web/test_web_search_provider_plugins.py` | Updated all 4 parametrize sections to include `"jina"`; renamed `test_all_seven_plugins_present_in_registry` → `test_all_eight_plugins_present_in_registry`; added capability flag test for jina (True, True, False) |

## Provider Capabilities

| Capability | Supported |
|------------|-----------|
| Search     | ✅        |
| Extract    | ✅        |
| Crawl      | ❌        |

## Configuration

```yaml
# ~/.hermes/config.yaml
web:
  search_backend: "jina"      # explicit per-capability
  extract_backend: "jina"     # explicit per-capability
  backend: "jina"             # shared fallback
```

```env
# ~/.hermes/.env
JINA_API_KEY=<your-key>       # required — get one at https://jina.ai
```

## Endpoint Details

| Endpoint | Method | Auth | Response |
|----------|--------|------|----------|
| `https://s.jina.ai/` | POST (query in body) | `Bearer {JINA_API_KEY}` | `text/plain` |
| `https://r.jina.ai/<url>` | GET | `Bearer {JINA_API_KEY}` | `text/plain` |

Search uses `numResults` param; Reader returns cleaned content with title extraction.

## Legacy Preference Order

Jina is placed after exa and before searxng in the auto-detect fallback order:

```
firecrawl → parallel → tavily → exa → jina → searxng → brave-free → ddgs
```

This positions Jina as a mid-tier paid provider — above free tiers but below the highest-tier options.

## Response Shapes

### Search
```json
{
  "success": true,
  "data": {
    "web": [
      {
        "title": "string",
        "url": "string",
        "description": "string",
        "position": 1
      }
    ]
  }
}
```

### Extract
Returns a bare list of result dicts per URL (matches all other providers):
```json
[
  {
    "url": "string",
    "title": "string",
    "content": "string",
    "raw_content": "string",
    "metadata": {}
  }
]
```

On per-URL failure, the result dict includes an `"error"` field.

## Response Parser

Jina's search endpoint returns text (not JSON), so `_parse_search_response()` uses heuristic regex:

1. Splits response by `---` separators (result boundaries)
2. Extracts URLs via `https?://[^\\s<>\"']+` pattern
3. Extracts titles via `^(?:Title|title)[:\\s]+(.+)$` pattern
4. Falls back to URL hostname for titles when no title line found
5. Cleans description by stripping title/snippet markers

On parse failure, returns `{"success": False, "error": "..."}` to trigger fallback to the next provider.

## Code Review Fixes

This PR incorporates 6 fixes identified during code review:

1. **Parse failure returns `{"success": False}`** — Previously returned `{"success": True, "data": [...]}` with garbage data on parse failure, masking the error and preventing fallback. Now returns `{"success": False, "error": "..."}` so the caller correctly falls back.

2. **API key safety gap** — Both `search()` and `extract()` used bare `os.environ['JINA_API_KEY']` dict access, which crashes with `KeyError` if the env var is missing. Changed to `os.environ.get()` with an early return and structured error message.

3. **`httpx.Client` connection pooling** — `extract()` created a new `httpx.Client` per URL (one per URL, each with its own connection pool). Now creates one client outside the URL loop and reuses it across all URLs, enabling proper connection pooling.

4. **`_extract_title()` markdown heading fallback** — Previously explicitly *skipped* lines starting with `#` (markdown heading markers), so Jina's markdown output titles were missed. Now *handles* them — strips the `#` markers before returning via `re.sub(r'^#{1,6}\s+', '', first)`.

5. **Search parser resilience** — Combined with fix #1, the parser degrades gracefully. The `---` section splitting handles structured responses; the exception handler catches format changes.

6. **ABC docstring extract shape** — The module-level docstring in `web_search_provider.py` documented extract returning `{"success": True, "data": [...]}` but the actual implementation (and all other providers) return a bare list. Corrected to match reality and added `jina` to the provider list.

## Known Limitations

- **Search parser**: The search response parser uses heuristic regex on raw text output. If Jina changes their response format, parsing may degrade silently. No unit tests currently cover this.
- **No crawl support**: Jina does not implement `supports_crawl()`.
- **Rate limits**: Jina free tier is 100 req/day.

## Testing

Manual testing performed:
- Search with various queries returns properly formatted results
- Extract with multiple URLs processes each independently
- Error handling returns proper failure dicts
- `is_available()` correctly detects missing API key
- Syntax check passes on both modified files

Automated tests:
- `tests/plugins/web/test_web_search_provider_plugins.py` — 15 tests pass (registry, capability flags, setup schema, error shapes)
- `tests/tools/test_web_providers.py` — 47 tests pass (ABC contracts, config, per-capability routing, error envelope parity)
- `scripts/check-windows-footguns.py` — no footguns in modified files
- Total: 62 tests, 0 failures

## How to Test

1. Set `JINA_API_KEY` in `~/.hermes/.env`
2. Set `web.search_backend: "jina"` (and/or `web.extract_backend: "jina"`) in `~/.hermes/config.yaml`
3. Trigger a web search — verify results appear in the response
4. Trigger a web extract on a URL — verify cleaned content appears
5. Remove the API key — verify the system falls back to the next provider without crashing

## Notes for Reviewers

1. The `_parse_search_response()` function is the most fragile part of this implementation. It uses heuristic regex on raw text output from Jina's search endpoint — if Jina changes their response format, parsing may degrade silently. Consider adding unit tests in a follow-up PR.
