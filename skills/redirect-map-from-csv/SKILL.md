---
name: redirect-map-from-csv
description: Turn a CSV of old→new URLs into BrightSite 301/302 redirects in one pass. Validates source paths, deduplicates against existing redirects, and reports collisions before creating anything. Use when the user mentions "redirect map," "import redirects," "redirects from spreadsheet," "migrate URLs," "bulk redirects," "301 list," or has a CSV with URL mappings.
---

# Redirect Map from CSV

Agencies live in spreadsheets. This skill turns a CSV into BrightSite redirects without the user clicking through the redirects UI 200 times.

## When to use this

- After a site migration where many URLs changed (use alongside `migrate-wordpress-blog`).
- After an info architecture restructure (collapsing /services/web/ /services/mobile/ /services/design/ into /services/).
- When consolidating multiple domains into one.

## Inputs you need from the user

1. **Account ID**
2. **CSV path or content** — required columns: `source_path`, `target_url`. Optional: `redirect_type` (301 or 302, default 301), `is_active` (default true), `notes`.
3. **What to do on conflict** — if a redirect already exists for a given `source_path`: skip, overwrite, or ask per row? Default: ask user once for the whole batch.

## CSV format

Minimum:

```csv
source_path,target_url
/old-about,/about
/services/web-design,/services
/blog/2019/old-post,/blog/new-post
```

Full:

```csv
source_path,target_url,redirect_type,is_active,notes
/old-about,/about,301,true,IA cleanup
/sale-2023,/promotions,302,true,temporary holiday redirect
/legacy-api,https://api.client.com,301,true,external redirect
```

`source_path` should always be a path starting with `/` (no domain). `target_url` can be a path or a full URL.

## Workflow

### Step 1: Parse, coerce, and validate

**Critical:** CSV parsers return everything as strings. The BrightSite MCP rejects string values for `redirect_type` (wants integer `301` or `302`) and `is_active` (wants boolean `true`/`false`). Coerce these BEFORE building any MCP payload — this is the #1 way the call fails silently:

- `redirect_type`: parse as integer. Pass `301`, not `"301"`.
- `is_active`: accept `true`/`false`/`yes`/`no`/`1`/`0` (case-insensitive) in the CSV, but pass real booleans `true`/`false` to the MCP.

Then for each row, validate:

- `source_path` starts with `/`
- `source_path` doesn't contain a domain
- `target_url` is a path or a valid URL
- `redirect_type`, if present, parses as `301` or `302`
- `source_path` doesn't appear twice in the CSV itself (deduplicate or warn)

If any row fails validation, stop and report the bad rows. Don't proceed until they're fixed.

### Step 2: Check for collisions

Call `mcp__brightsite__list_redirects` and build a set of existing `source_path` values. For each CSV row, check if `source_path` is already a redirect.

Report:
- `N` new redirects to create
- `M` conflicts with existing redirects
- For conflicts, show a sample (first 5) of `existing target → CSV target`

Ask the user: skip conflicts, overwrite them, or cancel?

### Step 3: Check target paths for circularity

For each `target_url` that's a relative path, check for self-redirects and loops. A → A is invalid. A → B and B → A is a loop. Longer cycles like A → B → C → A are also loops. A → B and B → C is a chain, not a loop; warn that chains add latency, but don't treat them as validation failures unless the user wants to flatten them.

### Step 4: Show a preview

Before creating anything, show the user the first 5 redirects you're about to create, formatted as:

```
/old-path → /new-path  (301)
/old-2    → /new-2     (301)
...
```

Wait for confirmation.

### Step 5: Create redirects

Iterate through rows. For each:

- New row: `mcp__brightsite__create_redirect` with the fields from the CSV.
- Conflict + user chose "overwrite": `mcp__brightsite__update_redirect` with the existing redirect ID.
- Conflict + user chose "skip": skip.

Before each MCP call, strip `notes` and any unknown CSV-only fields. Only pass `account_id`, `source_path`, `target_url`, `redirect_type`, and `is_active` (plus `id` for updates).

Throttle to ~10/sec. Report progress every 25 redirects.

### Step 6: Output a summary

```
Redirects created:  187
Redirects updated:   12
Redirects skipped:    3 (conflicts, user chose skip)
Errors:               0

Spot-check by visiting:
- {site_url}/old-about      → should land at /about
- {site_url}/services/web-design → should land at /services
- {site_url}/blog/2019/old-post  → should land at /blog/new-post
```

Suggest the user spot-checks 3-5 redirects in an incognito browser before assuming the import worked.

## Anti-patterns to avoid

- **Don't create redirects with `source_path` ending in a slash if the existing routes don't use trailing slashes** (or vice versa). Pick one convention and normalize. Ask the user if unsure.
- **Don't blindly overwrite existing redirects.** Conflicts often indicate the client made manual changes that matter. Always show the user the conflict before overwriting.
- **Don't create redirects that point at non-existent pages.** Optionally cross-check `target_url` against `mcp__brightsite__list_pages` and warn if a target slug doesn't exist. (This isn't a hard error — the target might be a non-BrightSite URL or a not-yet-created page.)

## Example invocation

> I have a CSV at `~/clients/acme/redirects.csv` with about 200 rows. Account ID `YOUR_ACCOUNT_ID`. Skip any conflicts with existing redirects.

## Tools used

- `mcp__brightsite__list_redirects` — collision check
- `mcp__brightsite__create_redirect` — new rows
- `mcp__brightsite__update_redirect` — overwrite on conflict
- `mcp__brightsite__list_pages` — optional target validation
