# Data sourcing — find it, then actually access it

Stage 3. Two jobs: confirm relevant data exists, and nail down the *real* access
mechanism by probing the live source. Most lost time here comes from assuming schemas,
encodings, and endpoint names instead of checking.

**Scaffold once, before pulling:** create `data/processed/` (committed aggregates + the Stage-6
table) and `scripts/` (for `preprocess.py`), and keep API keys in a **gitignored `.env`**. Raw
downloads are transient — aggregate them into `data/processed/` and delete the bulky raw.

## Find candidate data

- Search open-data portals and the web for datasets matching the goal's entities and
  granularity (e.g., per-station monthly, per-user daily). Prefer official/primary
  sources; note license and update cadence.
- For each candidate, record: what it provides, grain, time span, format(s), and how
  current it is. A dataset that's stale or too coarse for the goal is a dead end —
  catch that now.

## Probe the live source before trusting it

Write a tiny script that fetches a real sample and prints the schema. Confirm:

- **Exact field names + meanings** (not the doc's prose — the actual keys returned).
- **The join key** across datasets, and its format (zero-padding, prefixes, types).
  Normalize with `int()`/zfill as needed; verify a few rows actually match.
- **Encoding.** Korean public CSVs are often `cp949`/`euc-kr`, not UTF-8. `cp949` is a
  superset of `euc-kr` — try it when `euc-kr` throws `UnicodeDecodeError`.
- **Quoting quirks.** Mixed quoting (some values in `'...'`, comma-containing values in
  `"..."`) breaks naive parsing. Parse with the default quote char, then strip stray
  quotes; or use `engine="python"` with `on_bad_lines="skip"` and report what dropped.
- **Schema drift across files.** Multi-year exports often change column names/order/shape
  between periods. Prefer a contiguous span with a consistent schema; harmonize the rest
  by position, and **log what you excluded** rather than silently dropping it.

## Pull efficiently — bulk, not row-by-row

Get many records per request; don't fetch one record at a time.

- **Prefer, in order:** a bulk file / dump / full CSV download → a batch or paged API at the
  **largest page size** allowed → server-side **filters** (date range, region, category) so you
  pull only the rows you need. One query that returns thousands of rows beats thousands of queries.
- **Avoid the N+1 loop:** one HTTP request per record (fetching each id / day / station separately)
  is slow, hammers the source, and trips rate limits — it can turn minutes into hours.
- **Last resort is OK.** If the source genuinely exposes no bulk / batch / paged / filtered access,
  a per-record loop is acceptable — but then add a polite delay / backoff between calls, **cache
  every response** so you never re-pull, and fetch **only the subset you actually need**.

## API access patterns

- **Keys via `.env`.** Never hardcode. `from dotenv import load_dotenv; load_dotenv()`,
  then `os.getenv("KEY")`. Ensure `.env` is gitignored before any commit.
- **Paging.** Many open APIs cap rows per call (e.g., 1000). Loop with start/end indices
  until you reach `list_total_count`. Watch for the response wrapper key (it may not match
  the service name — inspect `next(iter(resp))`).
- **Service-name discovery.** If the portal page is JS-rendered and hides the OpenAPI
  service name, brute-probe a few candidate names live: a real service returns rows or a
  recognizable error; a wrong one returns a generic 500. Distinguish "wrong name" from
  "no access / needs param".

## Keep the notebook light

Large raw downloads (hundreds of MB) shouldn't live in the analysis notebook. Put
download + aggregation in a `scripts/preprocess.py`, write compact aggregates to
`data/processed/` (commit those), and have the notebook read the small processed files.
Delete bulky raw after aggregating. This keeps the notebook reproducible and fast, and
the repo small.

## Example: live schema probe (adapt freely)
```python
import os, json, urllib.request
from dotenv import load_dotenv
load_dotenv(); KEY = os.getenv("OPENAPI")
url = f"http://openapi.example/{KEY}/json/SERVICE/1/3/"
r = json.load(urllib.request.urlopen(url, timeout=30))
body = r[next(iter(r))]                 # wrapper key may differ from service name
print(body.get("RESULT"), list(body["row"][0].keys()))   # status + real columns
```
