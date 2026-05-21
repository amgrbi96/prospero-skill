# PROSPERO API Reference

Complete specification for the PROSPERO search API, reverse-engineered from the frontend JavaScript (`main.3a4214f2.js`).

## Endpoint

```
POST https://www.crd.york.ac.uk/PROSPERO/api/search
```

## Authentication

No API key required. Instead, generate a per-request token:

```
prospero-auth-token = base64Encode(currentTimeInMilliseconds)
```

Examples:
- JavaScript: `btoa(Date.now().toString())`
- Python: `base64.b64encode(str(int(time.time() * 1000)).encode()).decode()`

The token is a base64-encoded millisecond timestamp. Generate a fresh token for each request.

## Required Headers

```
Content-Type: application/json
Accept: application/json
Origin: https://www.crd.york.ac.uk
Referer: https://www.crd.york.ac.uk/prospero/search
prospero-auth-token: <generated token>
```

**Note**: The API will respond without the `prospero-auth-token` header, but including it is more reliable and consistent with the frontend's behavior.

## Request Body

```json
{
  "term": "search query terms",
  "page": 1,
  "nperpage": 20,
  "sort": "TI",
  "sortorder": "ASC",
  "filters": [],
  "download": false,
  "actual": "same as term"
}
```

| Field | Type | Description |
|---|---|---|
| `term` | string | Search query. Combine terms with AND/OR. |
| `actual` | string | Mirror of `term`. Appears to be used for highlighting/tracking. |
| `page` | integer | Page number (1-indexed) |
| `nperpage` | integer | Results per page (max appears to be ~100) |
| `sort` | string | Sort field: `TI` (title), `DA` (date), etc. |
| `sortorder` | string | `ASC` or `DESC` |
| `filters` | array | Server-side filters (see below) |
| `download` | boolean | `true` returns richer metadata per record |

### Sort options

| Code | Sort by |
|---|---|
| `TI` | Title |
| `DA` | Date |

### Filter format

Filters use a specific array-of-objects format where each object has `name` (string) and `value` (array of strings):

```json
{
  "filters": [
    {"name": "reviewstatus", "value": ["Ongoing"]},
    {"name": "recordtype", "value": ["Clinical"]}
  ]
}
```

**Common mistake**: The key is `name` (not `field`), and `value` must be an array (not a string). Using the wrong format silently returns unfiltered results.

### Available filters

| Filter name | Description | Example values |
|---|---|---|
| `reviewstatus` | Review status | `Ongoing`, `Completed` |
| `recordtype` | Type of record | `Clinical` |
| `yearfirstpublished` | Publication year | `2024`, `2025` |
| `region` | Geographic region | `Europe`, `North America` |
| `dateinprospero` | Date range in PROSPERO | `01 Jan 2024 to 01 Jan 2025` |
| `funders` | Funding body | `NIH`, `Wellcome Trust` |

The API translates these filters to Elasticsearch wildcard queries internally.

### Multi-filter example

```json
{
  "filters": [
    {"name": "reviewstatus", "value": ["Ongoing"]},
    {"name": "recordtype", "value": ["Clinical"]},
    {"name": "yearfirstpublished", "value": ["2024", "2025"]}
  ]
}
```

## Response Structure

The API returns an array with exactly one element containing an Elasticsearch-style response:

```json
[{
  "linenumber": 1,
  "term": "echoed search term",
  "hits": 3583,
  "retvals": {
    "hits": {
      "total": {"value": 3583, "relation": "eq"},
      "hits": [
        {
          "_source": {
            "title": "<HTML-wrapped title>",
            "reviewstatus": "Ongoing",
            "recordid": 1023387,
            "recordtype": "Clinical",
            "accessionnumber": "CRD420251023387",
            "yearfirstpublished": 2025
          }
        }
      ]
    },
    "aggs": {}
  }
}]
```

### Key response fields

| Path | Description |
|---|---|
| `[0].hits` | Total hit count (top-level shortcut) |
| `[0].retvals.hits.total.value` | Total hit count (Elasticsearch format) |
| `[0].retvals.hits.hits[]._source.title` | Review title (may contain HTML tags) |
| `[0].retvals.hits.hits[]._source.accessionnumber` | CRD accession number |
| `[0].retvals.hits.hits[]._source.reviewstatus` | `Ongoing` or `Completed` |
| `[0].retvals.hits.hits[]._source.recordid` | Numeric record ID |
| `[0].retvals.hits.hits[]._source.recordtype` | Record type (e.g., `Clinical`) |
| `[0].retvals.hits.hits[]._source.yearfirstpublished` | Publication year |

### Download mode (`download: true`)

Setting `download: true` returns additional `_source` fields including:
- Review question text
- Author list
- Registration and completion dates
- More detailed metadata

Use download mode when you need full PICOS comparison data.

## Individual record lookup

There is also a view endpoint for fetching a single record by CRD number:

```
GET https://www.crd.york.ac.uk/PROSPERO/api/view/<accessionnumber>
```

Returns `{"record": {...}, "json": [...]}` for valid records, or `{"record": {}, "json": []}` for non-existent ones.

## Pagination

For queries returning more results than `nperpage`, increment `page`:

```json
{"term": "diabetes", "page": 2, "nperpage": 20}
```

The `hits` field in the response tells you the total count. Calculate total pages as `Math.ceil(total / nperpage)`.

## Error responses

| Status | Meaning |
|---|---|
| 429 | Rate limited. Retry after a delay. |
| 5xx | Server error. Retry once. |
| Timeout | Network issue. The PROSPERO servers can be slow. |

The API does not appear to return structured error bodies — responses are typically empty or generic HTML on error.

## Complete request example

```python
import base64
import json
import time
import urllib.request

def search_prospero(query, filters=None, page=1, per_page=20, download=False):
    token = base64.b64encode(str(int(time.time() * 1000)).encode()).decode()
    body = json.dumps({
        "term": query,
        "page": page,
        "nperpage": per_page,
        "sort": "TI",
        "sortorder": "ASC",
        "filters": filters or [],
        "download": download,
        "actual": query,
    }).encode()

    req = urllib.request.Request(
        "https://www.crd.york.ac.uk/PROSPERO/api/search",
        data=body,
        method="POST",
    )
    req.add_header("Content-Type", "application/json")
    req.add_header("Accept", "application/json")
    req.add_header("Origin", "https://www.crd.york.ac.uk")
    req.add_header("Referer", "https://www.crd.york.ac.uk/prospero/search")
    req.add_header("prospero-auth-token", token)

    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.loads(resp.read().decode())

# Example: search for ongoing diabetes reviews
results = search_prospero(
    "type 2 diabetes AND metformin",
    filters=[{"name": "reviewstatus", "value": ["Ongoing"]}],
)
print(f"Total hits: {results[0]['hits']}")
for hit in results[0]["retvals"]["hits"]["hits"]:
    src = hit["_source"]
    print(f"  {src['accessionnumber']}: {src['title']}")
```
