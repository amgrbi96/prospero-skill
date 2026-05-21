---
name: prospero-search
description: Search the PROSPERO registry for registered systematic reviews and check for duplicate or overlapping reviews before registering a new one. Use when the user mentions PROSPERO, systematic review registration, checking for duplicate reviews, or asks whether their research question has already been studied — even if they don't explicitly say 'PROSPERO'. Also triggers on mentions of 'protocol registration', 'review overlap', 'CRD number lookup', or 'prospective register'. Useful for researchers planning systematic reviews, scoping reviews, or meta-analyses who need to verify novelty before registration.
---

# PROSPERO Systematic Review Search & Duplicate Check

Search the PROSPERO international prospective register of systematic reviews and detect whether a planned review duplicates existing registered reviews.

## When to use this skill

- A researcher wants to check if their planned systematic review overlaps with existing reviews
- Someone asks to search PROSPERO by keywords, status, year, region, or funder
- A user needs to verify novelty of a research question for systematic review registration
- Someone asks to look up a CRD accession number (e.g., CRD420251023387)
- Protocol registration preparation for Cochrane, JBI, or other SR methodologies

## How PROSPERO's API works

The PROSPERO search API is undocumented and was reverse-engineered from their frontend JavaScript. Read `references/api_reference.md` for the full specification including request/response formats, filter options, and authentication.

Key points:
- **Endpoint**: `POST https://www.crd.york.ac.uk/PROSPERO/api/search`
- **Auth**: Generate a fresh `prospero-auth-token` header for each request by base64-encoding the current millisecond timestamp (`btoa(Date.now().toString())` in JS, `base64.b64encode(str(int(time.time() * 1000)))` in Python)
- **Required headers**: `Content-Type: application/json`, `Accept: application/json`, `Origin: https://www.crd.york.ac.uk`, `Referer: https://www.crd.york.ac.uk/prospero/search`

## Workflow

### Step 1: Extract search terms

From the user's research question or protocol, extract the core **Population** and **Intervention** terms. Combine them as the search query, e.g. `"type 2 diabetes AND metformin"`.

If the user provides specific CRD numbers or a narrow query, use those directly.

### Step 2: Build and execute the search

Construct the API request. A minimal search looks like:

```json
{
  "term": "<search terms>",
  "page": 1,
  "nperpage": 20,
  "sort": "TI",
  "sortorder": "ASC",
  "filters": [],
  "download": false,
  "actual": "<same as term>"
}
```

**Filters** use a specific format (see `references/api_reference.md` for full details):

| Filter name | Values | Example |
|---|---|---|
| `reviewstatus` | `Ongoing`, `Completed` | `[{"name": "reviewstatus", "value": ["Ongoing"]}]` |
| `recordtype` | `Clinical` | `[{"name": "recordtype", "value": ["Clinical"]}]` |
| `yearfirstpublished` | Year strings | `[{"name": "yearfirstpublished", "value": ["2024", "2025"]}]` |
| `region` | Region names | `[{"name": "region", "value": ["Europe"]}]` |
| `dateinprospero` | Date range string | `[{"name": "dateinprospero", "value": ["01 Jan 2024 to 01 Jan 2025"]}]` |
| `funders` | Funder names | `[{"name": "funders", "value": ["NIH"]}]` |

For duplicate checking, filter to `Ongoing` reviews only — completed reviews are less concerning for registration purposes.

Use `download: true` to get richer metadata (review question, authors, dates) when you need detailed comparison.

### Step 3: Analyze results

Read the response structure from `references/api_reference.md`. Key fields:

- `hits.total.value` — total number of matching reviews
- `hits.hits[]._source.title` — review titles (HTML-wrapped)
- `hits.hits[]._source.accessionnumber` — CRD number
- `hits.hits[]._source.reviewstatus` — Ongoing / Completed

### Step 4: Duplicate / overlap assessment

For each PROSPERO result, compare against the proposed review using PICOS elements:

| Element | What to compare |
|---|---|
| **P**opulation | Same or overlapping patient group? |
| **I**ntervention | Same or overlapping treatment/exposure? |
| **C**omparator | Same or overlapping comparison? |
| **O**utcomes | Same or overlapping outcome measures? |
| **S**tudy design | Same review type (systematic, meta-analysis, scoping)? |

Score overlap as **matched elements / 5**. Flag reviews scoring **3 or higher** as potential duplicates.

### Step 5: Propose differentiations (when overlap detected)

When overlap >= 3/5, suggest specific PICOS refinements to differentiate the proposed review:

| Element | Narrowing strategy | Example |
|---|---|---|
| Population | Restrict age, setting, severity | Adults 18-65 -> community-dwelling adults 65+ |
| Intervention | Narrow dosage, delivery, duration | Exercise -> supervised resistance training 3x/week |
| Comparator | Add comparison the existing review lacks | Usual care -> usual care + attention control |
| Outcomes | Focus on different outcome types | Clinician-reported -> patient-reported outcomes |
| Study design | Restrict design type | Any design -> RCTs only |

Present the differentiated PICOS to the user for acceptance or further refinement.

### Step 6: Present results

Format the output clearly:

```
PROSPERO Search Results
=======================
Query: "<search terms>"
Filters applied: [list filters]
Total results: <count>

Potential Overlaps:
-------------------
1. [CRD#######] Title...
   Status: Ongoing
   PICOS Overlap: X/5
   Matched elements: [P, I, ...]
   Suggested differentiation: [specific changes]

Unique Review Candidates:
--------------------------
[CRD#######] Title... — no significant overlap

Conclusion:
-----------
[your assessment and recommendations]
```

## Error handling

| Error | Response |
|---|---|
| HTTP 429 (rate limited) | Wait 5 seconds, retry once. If still 429, report unavailability and suggest manual check at https://www.crd.york.ac.uk/prospero |
| HTTP 5xx | Retry once after 3 seconds. If still failing, report unavailability |
| Network timeout (>30s) | Report unavailability, suggest manual search |
| Invalid response | Log the issue, report what was received |

## Constraints

- **Rate limiting**: Send one request at a time. No parallel PROSPERO queries.
- **Max 3 refinement cycles**: After 3 rounds of differentiation attempts, ask the user explicitly before continuing.
- **Client-side PICOS extraction**: PROSPERO records have limited metadata in standard mode. Extract PICOS from title text + available fields. Use `download: true` for borderline cases.

## Bundled resources

- `references/api_reference.md` — Full PROSPERO API specification with request/response examples, filter format, pagination, and a complete Python code example. Read this before making API calls.
- `scripts/test_api.py` — Run `python3 scripts/test_api.py` to validate the API is reachable and see example responses.
