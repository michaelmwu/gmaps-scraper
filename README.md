# GMaps Scraper

Extract data from Google Maps saved lists and individual place pages.

The scraper fetches Google Maps URLs, reads runtime data or the rendered place
panel, and returns structured JSON.

## Requirements

- Python `3.14`
- `uv`
- `curl_cffi` for the primary fetch path
- `cloakbrowser` for browser mode and HTTP fallback

## Install

This project is intended to be consumed directly from source rather than from PyPI.

```bash
uv add git+https://github.com/michaelmwu/gmaps-scraper.git
```

If you vendor the package, also add the runtime dependency:

```bash
uv add curl-cffi cloakbrowser
```

## CLI

The package installs a `gmaps-scraper` command.

Basic usage:

```bash
uv run gmaps-scraper "https://maps.app.goo.gl/MG2Vd5pWBkL7hXL18"
```

Scrape a place page:

```bash
uv run gmaps-scraper \
  "https://www.google.com/maps/place/Den/@35.6731762,139.7127216,17z" \
  --kind place
```

Scrape a place page and download its representative image locally:

```bash
uv run gmaps-scraper \
  "https://www.google.com/maps/place/Den/@35.6731762,139.7127216,17z" \
  --kind place \
  --download-photo den-photo.jpg
```

Scrape a place page and download the main place photo specifically:

```bash
uv run gmaps-scraper \
  "https://www.google.com/maps/place/Den/@35.6731762,139.7127216,17z" \
  --kind place \
  --download-main-photo den-main-photo.jpg
```

Optionally allow LLM repair for thin or suspicious place results:

```bash
OPENAI_API_KEY=...
LLM_MODEL=gpt-5-mini

uv run gmaps-scraper \
  "https://www.google.com/maps/place/Den/@35.6731762,139.7127216,17z" \
  --kind place \
  --llm-repair
```

The LLM path uses an OpenAI-compatible chat completions endpoint only when
explicitly enabled. Deterministic DOM and preview extraction still run first.
Configuration precedence is:

1. Checked-in app defaults
2. Worktree-local `llm.local.json`
3. Environment variables

Built-in aliases currently include `gpt-5-mini`, `gpt-4o-mini`,
`gpt-4.1-mini`, `haiku`, and `sonnet`.

Example worktree-local `llm.local.json` for Fireworks:

```json
{
  "models": {
    "kimi-k2p6": {
      "provider": "fireworks",
      "model": "accounts/fireworks/models/kimi-k2p6",
      "omit_temperature": true,
      "request_options": {
        "reasoning_effort": "low",
        "max_tokens": 512
      }
    }
  }
}
```

Environment variables can override the local file. Useful knobs include
`LLM_MODEL`, `LLM_PROVIDER`, `LLM_BASE_URL`, `LLM_API_KEY`,
`LLM_MAX_TOKENS`, `LLM_TEMPERATURE`, `LLM_REASONING_EFFORT`, and
`LLM_OMIT_TEMPERATURE`.

Explicit fetch modes:

```bash
uv run gmaps-scraper URL --fetch-mode auto
uv run gmaps-scraper URL --fetch-mode curl
uv run gmaps-scraper URL --fetch-mode browser
```

Mode behavior:

- `auto` uses `curl_cffi` first and falls back to the browser if parsing fails
- `curl` uses only the HTTP path
- `browser` uses only the browser path

Write JSON to a file:

```bash
uv run gmaps-scraper \
  "https://maps.app.goo.gl/MG2Vd5pWBkL7hXL18" \
  --output saved-list.json
```

Run with a visible browser for debugging:

```bash
uv run gmaps-scraper \
  "https://maps.app.goo.gl/MG2Vd5pWBkL7hXL18" \
  --headed
```

Write place debug artifacts and a compact selector recipe:

```bash
uv run gmaps-scraper URL \
  --kind place \
  --debug-output-dir .context/places/example
```

Reuse a shared browser profile across worktrees:

```bash
uv run gmaps-scraper URL \
  --kind place \
  --session-dir "$CONDUCTOR_ROOT_PATH/.gmaps-scraper/session"
```

Batch scrape places with warmed browser contexts, per-place retries, and staggered starts.
With `--max-concurrency 1`, the run reuses one browser profile. With higher
concurrency, each worker gets its own profile under `--session-dir`.

`places.txt` is newline-delimited:

```text
https://www.google.com/maps/place/Den/@35.6731762,139.7127216,17z
# blank lines and comments are ignored
https://www.google.com/maps/place/Narisawa/@35.6724929,139.7111143,17z
```

```bash
uv run gmaps-scraper \
  --kind place \
  --input places.txt \
  --session-dir "$CONDUCTOR_ROOT_PATH/.gmaps-scraper/session" \
  --max-concurrency 1 \
  --max-retries 2 \
  --retry-backoff-ms 2000 \
  --stagger-ms 500 \
  --screenshot-output-dir .context/places/screenshots \
  --output place-results.json
```

You can also pass multiple URLs directly or pipe them on stdin:

```bash
uv run gmaps-scraper --kind place URL1 URL2 URL3
printf '%s\n' URL1 URL2 URL3 | uv run gmaps-scraper --kind place --input -
```

Cache optional LLM repairs so unchanged evidence does not call the model again:

```bash
uv run gmaps-scraper \
  --kind place \
  --input places.txt \
  --llm-repair \
  --llm-cache-dir "$CONDUCTOR_ROOT_PATH/.gmaps-scraper/llm-cache" \
  --session-dir "$CONDUCTOR_ROOT_PATH/.gmaps-scraper/session"
```

When `--llm-cache-dir` is used, the scraper also stores exact, typed
translation-memory entries learned from LLM `address_display_en` and
`category_display_en` repairs in `translation-memory.learned.json`. Those entries
are reused from the same cache directory on later runs before calling the LLM,
but only for category labels, city/country/neighborhood components, floor/building
suffixes, and known address tokens. Reviews and review topics are never
translated or learned into this cache.

Maintainers can inspect and promote stable local entries into the bundled approved
memory through a normal PR:

```bash
uv run python scripts/promote_translation_memory.py \
  "$CONDUCTOR_ROOT_PATH/.gmaps-scraper/llm-cache/translation-memory.learned.json" \
  --dry-run
```

Downstream users can use the same workflow to contribute reusable mappings
upstream. Run production or enrichment jobs with a stable `--llm-cache-dir`, inspect
the generated `translation-memory.learned.json`, run the promotion script with
`--dry-run`, then open a PR containing the approved-memory diff and representative
tests. Upstream memory PRs should stay limited to category labels,
city/country/neighborhood components, floor/building suffixes, and known address
tokens. Do not contribute learned entries for reviews, review topics, venue names
by default, editorial text, or long full-address rewrites that are not clearly
safe component replacements.

Approved memory also supports typed pattern entries for structural address
tokens. Pattern templates intentionally support only numbered capture
substitution such as `{1}` and `{2}`:

```json
{
  "kind": "pattern",
  "source_pattern": "(?<![A-Za-z0-9])(\\d+)\\s*樓\\s*之\\s*(\\d+)",
  "target_template": "{1}F-{2}",
  "field_kinds": ["address_component"],
  "source_method": "approved",
  "confidence": "high"
}
```

Available CLI options:

- `--kind {list,place}` selects which scraper to run
- `--output PATH` writes the JSON result to a file
- `--download-photo PATH` saves the place photo to a local file when scraping a place
- `--download-main-photo PATH` saves the main place photo to a local file when available
- `--llm-repair` enables optional place-field repair with checked-in `llm.json`, optional `llm.local.json`, and env overrides
- `--llm-policy {never,on_quality_failure,always}` controls when the repair callback runs
- `--llm-env-file PATH` reads LLM settings from a dotenv-style file
- `--llm-cache-dir PATH` caches optional place-field repairs by URL, place ID, evidence hash, prompt version, and model namespace
- `--input PATH` reads place URLs from a newline-delimited file; use `--input -` for stdin
- `--max-concurrency INTEGER` controls batch place workers; each parallel worker gets its own browser context and profile subdirectory
- `--max-retries INTEGER` controls per-place retries for batch failures and retryable quality flags
- `--retry-backoff-ms INTEGER` controls linear retry backoff in batch place scraping
- `--stagger-ms INTEGER` delays batch place starts to avoid tight request bursts
- `--screenshot-output-dir PATH` writes overview and reviews place-page screenshots to a directory
- `--debug-output-dir PATH` writes list or place debug artifacts to a directory
- `--dump-debug-output` writes debug artifacts under `.gmaps-debug`
- `--headed` runs the browser in headed mode
- `--fetch-mode {auto,curl,browser}` selects the transport path
- `--session-dir PATH` reuses a persistent browser profile for browser fetches
- `--http-cookie-jar PATH` persists curl cookies across fetches
- `--proxy URL` sends curl and browser traffic through a proxy
- `--timeout-ms INTEGER` controls the overall fetch timeout
- `--settle-ms INTEGER` adds extra browser-only wait time after the page loads

## Library Usage

Import the package directly in application code:

```python
from pathlib import Path

from gmaps_scraper import (
    BrowserSessionConfig,
    HttpSessionConfig,
    cached_place_repairer,
    openai_compatible_place_repairer_from_env,
    scrape_place,
    scrape_places,
    scrape_saved_list,
)

result = scrape_saved_list(
    "https://maps.app.goo.gl/MG2Vd5pWBkL7hXL18",
    browser_session=BrowserSessionConfig(
        profile_dir=Path(".gmaps-scraper/session"),
    ),
    http_session=HttpSessionConfig(
        cookie_jar_path=Path(".gmaps-scraper/http-cookies.txt"),
    ),
)
place = scrape_place("https://www.google.com/maps/place/Den/@35.6731762,139.7127216,17z")
repaired_place = scrape_place(
    "https://www.google.com/maps/place/Den/@35.6731762,139.7127216,17z",
    llm_fallback=cached_place_repairer(
        openai_compatible_place_repairer_from_env(),
        cache_dir=Path(".gmaps-scraper/llm-cache"),
        cache_namespace="gpt-5-mini",
    ),
)
batch_results = scrape_places(
    [
        "https://www.google.com/maps/place/Den/@35.6731762,139.7127216,17z",
        "https://www.google.com/maps/place/Narisawa/@35.6724929,139.7111143,17z",
    ],
    browser_session=BrowserSessionConfig(
        profile_dir=Path(".gmaps-scraper/session"),
    ),
    max_retries=2,
    stagger_ms=500,
)

print(result.list_id)
print(result.resolved_url)
print(result.title)
print(result.to_dict())
print(place.review_count)
print(repaired_place.diagnostics)
print([item.to_dict() for item in batch_results])
```

Public top-level imports intended for consumers:

- `BrowserProxyConfig`
- `BrowserSessionConfig`
- `HttpSessionConfig`
- `scrape_saved_list`
- `scrape_place`
- `scrape_places`
- `collect_place_snapshot`
- `cached_place_repairer`
- `default_place_selector_recipe`
- `load_place_selector_recipe`
- `openai_compatible_place_repairer_from_env`
- `parse_saved_list_artifacts`
- `ListOwner`
- `SavedList`
- `Place`
- `PlaceAboutItem`
- `PlaceAboutSection`
- `PlaceDetails`
- `PlaceReview`
- `PlaceScrapeResult`
- `ReviewTopic`
- `PlaceExtractionDiagnostics`
- `PlaceLLMRepairRequest`
- `ParseError`
- `ScrapeError`
- `LLMRepairError`
- `write_default_place_selector_recipe`

## Output

A saved list result looks like this:

```json
{
  "source_url": "https://maps.app.goo.gl/MG2Vd5pWBkL7hXL18",
  "resolved_url": "https://www.google.com/maps/@30.5370705,125.4120472,6z/data=!4m3!11m2!2sUGEPbA20Qd-OH4uoWjmDgQ!3e3?entry=ttu",
  "list_id": "UGEPbA20Qd-OH4uoWjmDgQ",
  "title": "Tokyo Dinners",
  "description": "Best spots in the city",
  "owner": {
    "name": "Michael Wu",
    "photo_url": "https://lh3.googleusercontent.com/a-/ALV-UjW_i8-Eyr6conUhZ6tzGGlFe76mQTGeURI9NKDlca0FzlN0GY0Kjg",
    "profile_id": "104356373423434804635"
  },
  "collaborators": [
    {
      "name": "Micca Guan",
      "photo_url": "https://lh3.googleusercontent.com/a-/ALV-UjW_collaborator",
      "profile_id": "107609938540508038600"
    }
  ],
  "places": [
    {
      "name": "Yakumo",
      "address": "Shibuya, Tokyo",
      "note": "Delicious wonton ramen. You can ask for a mix of white and dark broth.",
      "is_favorite": true,
      "lat": 35.6501307,
      "lng": 139.6868459,
      "maps_url": "https://www.google.com/maps/search/?api=1&query=Yakumo%2C+Shibuya%2C+Tokyo",
      "added_by": {
        "name": "Micca Guan",
        "profile_id": "107609938540508038600"
      }
    }
  ]
}
```

`source_url` preserves the caller's input URL. `resolved_url` captures the final URL
after redirects, which is useful for short `maps.app.goo.gl` links.

For place pages, the scraper returns a `PlaceDetails` object with fields such as
`name`, `category`, `rating`, `review_count`, `address`, `status`, `website`,
`phone`, `plus_code`, `price_range`, `main_photo_url`, `photo_url`, review topic
chips, visible review snippets, About-tab sections, diagnostics, and coordinates
when available. The raw Google
address and category are preserved as `address` and `category`; when either
contains non-English script components and the scraper can safely normalize them,
it also returns `address_display_en` or `category_display_en` plus source/confidence
metadata.

`price_range` is Google Maps' raw display string, such as `SGD 100+`,
`NT$2,000+`, `¥10,000+`, or `$$`; the scraper does not convert it into a
currency-normalized budget. `about_sections` groups visible About-tab attributes
by Google section title, such as `Accessibility` or `Service options`.

Review topic chip `count` values are review mention counts. For example,
`{"label": "pho", "count": 24}` means Google exposed a `pho` topic chip mentioning
24 reviews.

## Behavior Notes

- Saved lists default to `curl_cffi` against Google Maps' preloaded XSSI endpoints.
- `--settle-ms` only affects browser fetches. `--timeout-ms` applies to both browser and curl.
- Reuse `HttpSessionConfig(cookie_jar_path=...)` or `--http-cookie-jar` when you want curl
  fetches to carry cookies across runs.
- Place pages currently use the browser path and extract review metadata from the
  rendered DOM.
- Place scraping also collects compact sanitized DOM evidence internally for optional
  LLM repair. Callers own model choice, credentials, budgets, and caching.
- Review topic chips are returned only when the rendered Maps page exposes them.
  The scraper clicks the Reviews tab and expands `+N` topic groups when present.
  Limited-view pages may omit reviews and review chips entirely.
- `reviews` contains the visible review snippets loaded in the rendered panel; it
  is not an exhaustive crawl of every review.
- Place debug dumps write raw investigation artifacts under `artifacts/` and a
  compact reusable `selector-recipe.json`. Reuse selector recipes, not full DOM
  snapshots, across sessions.
- `main_photo_url` is the direct main place photo when the rendered DOM exposes one.
- `photo_url` remains the best available image and falls back to the page's
  representative Maps image when the main photo isn't available.
- Browser automation remains available for debugging, consent flows, and fallback.
- By default each scrape uses a fresh browser session. Reuse a profile directory only
  when you want cookies, localStorage, and other browser state to persist across runs.
- Session rotation, clearing blocked profiles, and coordinating proxies across many
  scraping identities are caller-level policy decisions. The library only exposes the
  browser profile and proxy primitives needed to implement that policy.
- Parsing is defensive and tolerates partial metadata, but Google can change its runtime
  schema at any time.
- The parser prefers the explicit placelist ID from the resolved URL when available.

## Development

Install the dev environment:

```bash
uv sync --dev
```

Run the quality gates:

```bash
./scripts/lint.sh
./scripts/typecheck.sh
```

Install the git hooks locally:

```bash
uv run prek install
```

Run the same checks on demand:

```bash
uv run prek run --all-files
```

Run tests:

```bash
uv run python -m unittest discover -s tests
```
