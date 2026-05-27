# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A monorepo of Claude Code skills. Each skill lives in its own directory under `skills/`.

## Repository Structure

```
skills/
  prospero-search/       ← PROSPERO systematic review search & duplicate check
    SKILL.md             ← Skill definition (frontmatter + workflow)
    references/          ← API specs and supporting docs
    scripts/             ← Validation scripts
  <future-skill>/        ← Add new skills as siblings
```

- **`CLAUDE.md`** (this file) — repo-level development guidance, not distributed with any skill.
- Each skill folder is self-contained and independently installable.

## Testing

```bash
# PROSPERO skill — validate API connectivity (hits a live server)
python3 skills/prospero-search/scripts/test_api.py
```

No unit test suite, no build step, no linting. The only validation is the live API test above.

## Key Technical Details

- **Auth**: PROSPERO has no API key system. Instead, generate a per-request `prospero-auth-token` header by base64-encoding the current millisecond timestamp.
- **Filter format**: Filters use `{"name": "...", "value": [...]}` objects. The API converts them into query term modifications using hidden field codes (`rs` for reviewstatus, `rt` for recordtype, `yr` for year, `re` for region, `fi` for funders). Using the wrong key name or passing a string instead of array silently returns unfiltered results. The `dateinprospero` filter does not work at the API level.
- **Rate limiting**: The API rate-limits aggressively (HTTP 429). The skill must send one request at a time with retries.
- **`download: true`** mode only adds an `ris` (RIS citation format) field per record — not useful for PICOS extraction. Use the `/api/view/<CRD>` endpoint instead for full review content.

## Modifying Skills

When editing any `SKILL.md`, note:
- The frontmatter `description` field controls when the skill triggers — keep it comprehensive with synonyms and use-case phrases.
- The workflow is linear (6 steps). Maintain this structure; agents follow it sequentially.
- References to `references/` files are intentional — detailed specs live there to keep `SKILL.md` focused on workflow logic.

## Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md` with frontmatter (`name`, `description`) and workflow instructions.
2. Add supporting files (`references/`, `scripts/`) inside the skill directory.
3. Update this `CLAUDE.md` if the new skill has testing or setup requirements.
