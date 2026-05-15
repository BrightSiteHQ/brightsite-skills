---
name: generate-location-pages
description: Programmatic SEO. Given a CSV of locations (cities, neighborhoods, regions) and a page template, create one BrightSite draft page per row with location-specific copy, meta tags, and slugs. Use when the user mentions "location pages," "pSEO," "programmatic SEO," "city pages," "service area pages," "[service] in [city] pages," or wants to generate landing pages at scale from data.
---

# Generate Location Pages

Create one BrightSite draft page per row in a location CSV. Each page targets a `[service] in [city]` style keyword and shares a common template with per-location substitutions.

## When to use this

- Agency has a client with a service that varies by geography (HVAC, plumbing, dentists, real estate, legal, fitness).
- Client wants to rank for `[service] in [city]` across dozens or hundreds of locations.
- You have, or can produce, a CSV with one row per target location.

Do NOT use this for fewer than ~10 locations — at that scale, hand-built pages convert better. Use it when scale is the point.

## Inputs you need from the user

Ask the user for each of these before generating anything:

1. **Account ID** — their BrightSite account ID (or confirm which account they're authenticated as).
2. **CSV path or content** — at minimum, columns for `city` and `slug`. Encourage extra columns: `state`, `population`, `nearest_highway`, `local_landmark`, `phone`, `address`. More columns = less templated-feeling output.
3. **Service / offer** — what the client does ("emergency plumbing," "Pilates classes," "personal injury law").
4. **Template page** — either a page ID to clone, or HEEx + a description of the structure. If the user has no template, offer to draft one and confirm before generating at scale.
5. **Layout ID** (optional) — if the client has a layout other than the default.
6. **Draft mode confirmation** — this skill creates drafts only. The user can publish reviewed pages later with the BrightSite UI or `publish_page`.

## Workflow

### Step 1: Confirm scope before generating anything

Show the user:
- How many pages you will create
- The first row of the CSV
- A preview of the rendered HEEx, meta title, meta description, and slug for that first row
- The URL path each page will live at — `mcp__brightsite__create_page` returns a `path` field on the created page (e.g. `/emergency-plumbing-austin`); use that directly rather than reconstructing from the slug

**Wait for explicit "yes, generate them" before calling any `mcp__brightsite__create_page`.** Generating 200 pages by mistake is hard to undo.

### Step 2: Generate a single sample first

Call `mcp__brightsite__create_page` for row 1 only, with `status: "draft"`. The response includes a `path` field (e.g. `/emergency-plumbing-austin`); return that path to the user. Ask them to review it in the BrightSite editor and confirm before continuing.

### Step 3: Generate the rest

After approval, iterate through remaining rows. For each row, call `mcp__brightsite__create_page` with:

- `account_id`
- `title` — e.g. `"Emergency Plumbing in {city}, {state}"`
- `slug` — kebab-case, e.g. `"emergency-plumbing-{city-slug}"`
- `heex` — template with per-row substitutions
- `meta_title` — under 60 chars, e.g. `"Emergency Plumbing in {city} | {brand}"`
- `meta_description` — under 155 chars, action-oriented, includes the city name
- `status` — always `"draft"`. Do not publish from this skill.
- `layout_id` — if provided

Throttle to ~5 pages/sec to avoid rate limits. Report progress every 10 pages.

### Step 4: Output a summary

After the batch completes, report:
- Total pages created (and failures, if any)
- Total drafts created
- A list of created URLs the user can spot-check
- Suggested next step: run the `bulk-seo-audit` skill on the new pages, then publish reviewed pages via the BrightSite UI or `publish_page`.

## Anti-patterns to avoid

- **Don't generate pages with identical body content and only the city name swapped.** Google de-indexes thin pSEO. Use at least 2-3 location-specific substitutions per page (landmark, population, neighborhood, local stat).
- **Don't fabricate local facts.** If the user didn't provide a landmark column, leave that section out — don't hallucinate "Just minutes from [made-up landmark]."
- **Don't publish 50+ pages at once on a new domain.** Suggest the user publishes in batches over a few days so the sitemap submission feels natural.
- **Don't skip the meta description.** A blank meta description on a pSEO page is a wasted ranking opportunity. Generate one per page.

## Example invocation

> I have a CSV at `~/clients/acme-hvac/locations.csv` with columns `city,state,slug,population,landmark`. The client is Acme HVAC and they do emergency furnace repair. Account ID is `YOUR_ACCOUNT_ID`. Generate location pages, draft mode, using the template at page ID `8fd2k3jq91pn`.

## Tools used

- `mcp__brightsite__get_page` — read the template page
- `mcp__brightsite__create_page` — create each location page
- `mcp__brightsite__list_pages` — sanity-check that slugs don't collide with existing pages
