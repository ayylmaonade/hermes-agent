# Patch Plan — Jina Provider Fixes

All 6 fixes have been applied. See PR-DRAFT-JINA.md for full details.

## Fix 1: `_parse_search_response()` exception handler

**Problem**: On parse failure, returned `{"success": True, "data": [...]}` with garbage data — masking the error and preventing fallback.

**Fix**: Changed to `{"success": False, "error": "..."}` so the caller knows to fall back to the next provider.

**File**: `plugins/web/jina/provider.py` (line ~196)

## Fix 2: API key safety gap

**Problem**: `os.environ['JINA_API_KEY']` bare dict access crashes with `KeyError` if env var is missing.

**Fix**: Both `search()` and `extract()` now use `os.environ.get()` with an early return and structured error message.

**File**: `plugins/web/jina/provider.py` (lines ~61, ~83)

## Fix 3: `httpx.Client` connection pooling

**Problem**: `extract()` created a new `httpx.Client` per URL — one per URL, each with its own connection pool.

**Fix**: Moved `httpx.Client` creation outside the URL loop. One client reused across all URLs.

**File**: `plugins/web/jina/provider.py` (line ~89)

## Fix 4: `_extract_title()` markdown heading fallback

**Problem**: Explicitly skipped lines starting with `#` (markdown heading markers), so Jina's markdown output titles were missed.

**Fix**: Now handles `#` lines — strips the `#` markers before returning via `re.sub(r'^#{1,6}\s+', '', first)`.

**File**: `plugins/web/jina/provider.py` (lines ~213-215)

## Fix 5: Search parser resilience

**Problem**: No graceful degradation when parsing fails entirely.

**Fix**: Combined with Fix 1 — the parser degrades gracefully via the exception handler. The `---` section splitting handles structured responses; the exception handler catches format changes.

**File**: `plugins/web/jina/provider.py` (line ~196)

## Fix 6: ABC docstring

**Problem**: Module-level docstring in `web_search_provider.py` documented extract returning `{"success": True, "data": [...]}` but actual implementation returns a bare list.

**Fix**: Corrected module-level docstring to match reality. Also added `jina` to the provider list.

**File**: `agent/web_search_provider.py` (lines ~36-45)
