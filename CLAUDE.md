# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Claude Code skill for searching the [PROSPERO](https://www.crd.york.ac.uk/prospero) international prospective register of systematic reviews and detecting duplicate/overlapping reviews before new registration. The skill uses an undocumented, reverse-engineered PROSPERO API.

## Repository Structure

- **`SKILL.md`** — The skill definition. Contains frontmatter (name, description for trigger matching) and the full workflow instructions. This is the primary file — agents load it when the skill activates.
- **`references/api_reference.md`** — Complete PROSPERO API specification: endpoint, auth token generation, request/response formats, filters, pagination, and a full Python example. Read before making any API changes.
- **`scripts/test_api.py`** — Validation script that exercises the PROSPERO API with different configurations (token vs no-token, download mode, filters). No test framework — plain assertions via stdout.

## Testing

```bash
# Validate API connectivity and response parsing (hits a live server)
python3 scripts/test_api.py
```

No unit test suite, no build step, no linting. The only validation is the live API test above.

## Key Technical Details

- **Auth**: PROSPERO has no API key system. Instead, generate a per-request `prospero-auth-token` header by base64-encoding the current millisecond timestamp.
- **Filter format**: Filters use `{"name": "...", "value": [...]}` objects. The API converts them into query term modifications using hidden field codes (`rs` for reviewstatus, `rt` for recordtype, `yr` for year, `re` for region, `fi` for funders). Using the wrong key name or passing a string instead of array silently returns unfiltered results. The `dateinprospero` filter does not work at the API level.
- **Rate limiting**: The API rate-limits aggressively (HTTP 429). The skill must send one request at a time with retries.
- **`download: true`** mode only adds an `ris` (RIS citation format) field per record — not useful for PICOS extraction. Use the `/api/view/<CRD>` endpoint instead for full review content.

## Modifying the Skill

When editing `SKILL.md`, note:
- The frontmatter `description` field controls when the skill triggers — keep it comprehensive with synonyms and use-case phrases.
- The workflow is linear (6 steps). Maintain this structure; agents follow it sequentially.
- References to `references/api_reference.md` are intentional — detailed API specs live there to keep `SKILL.md` focused on workflow logic.
