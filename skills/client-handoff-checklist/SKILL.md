---
name: client-handoff-checklist
description: Pre-launch QA pass for a BrightSite site before handing it to a client or going live. Verifies site identity, error pages, blog settings, forms, redirects, analytics, and meta defaults are all configured. Outputs a checklist with green/yellow/red status. Use when the user mentions "client handoff," "pre-launch checklist," "launch QA," "going live," "ready to ship," "is this site ready," or wants a final QA pass before publishing a site.
---

# Client Handoff Checklist

Every agency forgets something. This is the "before you push the launch button" QA pass for a BrightSite site. Run it the day before going live.

## When to use this

- About to point the domain at a BrightSite site.
- About to email the client login credentials.
- Just finished a site build and want to know what you missed.

## Inputs you need from the user

1. **Account ID**
2. **Public URL** — what URL will visitors hit? (Used to validate canonical URLs.)
3. **Anything skip-worthy?** — e.g. "no blog on this site, skip blog checks" or "no contact form, skip form checks."

## Workflow

Run through every check below. For each, report `[OK]`, `[WARN]`, or `[FAIL]` and a one-line summary. Use plain text markers — no emojis unless the user explicitly asked for them.

### 1. Site identity (`mcp__brightsite__get_site_identity`)

- `[FAIL]` if `title` is empty or still "Untitled Site" / default
- `[FAIL]` if `logo_id` is null
- `[WARN]` if `icon_id` (favicon) is null
- `[FAIL]` if `business_type` is null (hurts schema/structured data)
- `[WARN]` if `business_description` is missing
- `[WARN]` if `email` or `phone` is missing (clients usually want at least one contact method)
- For local businesses: `[WARN]` if address fields are missing or `geo_lat`/`geo_lng` is null
- `[WARN]` if `social_profiles` is empty

### 2. Pages (`mcp__brightsite__list_pages` + `mcp__brightsite__get_page` on each)

- `[FAIL]` if no page has `is_home_page: true`
- `[FAIL]` if any page has `status` of `draft` and is linked from the homepage (sanity-check by scanning the home page HEEx for internal links)
- `[WARN]` if any page has `has_unpublished_changes: true` — those staged changes won't be live
- `[FAIL]` for any page missing `meta_title` or `meta_description`
- `[WARN]` for pages where `canonical_url` is set to a different domain than the public URL

Run a mini-version of `bulk-seo-audit` inline — count and report total SEO issues across all pages.

### 3. Error pages (`mcp__brightsite__list_error_pages`)

The returned rows expose the HTTP status as `status_code` (integer), so filter on that field.

- `[FAIL]` if no row with `status_code: 404` exists
- `[WARN]` if no row with `status_code: 500` exists
- `[WARN]` if any error page has `has_unpublished_changes: true`

### 4. Blog settings (`mcp__brightsite__get_blog_settings`) — skip if user said no blog

- `[OK]` to skip if `enabled: false` and user confirmed no blog
- `[FAIL]` if `enabled: true` but `blog_title` is empty
- `[WARN]` if `blog_url` or `listing_url` are missing or do not start with `/`. These are path settings, not full domains.
- `[WARN]` if `posts_per_page` is null or absurd (< 3 or > 50)
- `[WARN]` if `enable_open_graph` is `false` (poor social sharing)

### 5. Posts (`mcp__brightsite__list_posts` + `mcp__brightsite__get_post` on each) — skip if no blog

- `[WARN]` if zero published posts (clients usually want at least 1-3 launch posts)
- `[WARN]` if any post has `has_unpublished_changes: true`
- Inline mini-audit: fetch each post with `mcp__brightsite__get_post`, then count posts missing `meta_description` or both `feature_image_url` and `feature_image_id`

### 6. Forms (`mcp__brightsite__list_forms`) — skip if user said no forms

- `[WARN]` if no forms exist but the homepage contains "contact" copy (probably an oversight).
- `[OK]` if forms exist.
- `[INFO]` Always print this reminder: the MCP cannot inspect form delivery/recipient configuration. Tell the user: **"I confirmed forms exist, but you must verify in the BrightSite dashboard that each form has a notification email or webhook configured. A form with no recipient silently drops submissions."**

### 7. Redirects (`mcp__brightsite__list_redirects`)

- `[OK]` informational only — count how many redirects exist
- If the site was a migration, `[WARN]` if zero redirects exist (probably forgot the redirect map)

### 8. Analytics (`mcp__brightsite__get_analytics` with `metric: "summary"`, `period: "7d"`)

- `[OK]` if any data exists (means tracking is firing)
- `[WARN]` if `total_page_views: 0` over the last 7 days — either it's pre-launch (fine) or tracking isn't wired up (bad)

### 9. Global code (`mcp__brightsite__get_global_code`)

- `[WARN]` if `css_staged`, `js_staged`, or `tailwind_config_staged` differ from published versions (staged but unpublished global code)
- `[OK]` otherwise

### 10. Production-readiness final pass

- `[FAIL]` if any page or post still has placeholder text. Reuse the `heex`/`heex_staged` (pages) and `content`/`content_staged` (posts) strings already fetched in steps 2 and 5 — do NOT call any write tools. Scan BOTH live and staged versions; staged placeholder text will go live the next time the user publishes. Substrings to flag (case-insensitive): `"lorem ipsum"`, `"TODO"`, `"FIXME"`, `"test test"`, `"placeholder"`, `"xxx"` as a standalone token. Report the entity ID, which version (live vs staged), and the matched substring so the user can locate it in the editor.

## Output format

End with a clear summary block:

```
Handoff readiness: NOT READY
-----------------------------
[FAIL] 3 critical issues
[WARN] 8 warnings
[OK]  31 checks passed

Critical issues to fix before launch:
1. Home page has no meta_description (page ID 8fd2k3jq91pn)
2. 404 error page does not exist
3. Site identity missing logo

Warnings worth addressing:
- 2 pages have unpublished staged changes
- Blog has no posts published yet
- ...

Re-run this checklist after fixing the critical issues.
```

Or, on a clean site:

```
Handoff readiness: READY
-------------------------
[OK] 41 checks passed
[WARN] 2 warnings (non-blocking)

Non-blocking warnings:
- No social_profiles set in site identity
- Blog has only 1 published post

Ship it.
```

## Anti-patterns to avoid

- **Don't fix things automatically.** This is a checklist, not a remediation tool. The user wants to see what's wrong and decide.
- **Don't grade on a curve.** A site missing a meta_description on the homepage is `[FAIL]`, even if the rest is great. Be honest.
- **Don't skip checks the user didn't explicitly skip.** If the user didn't say "no blog," run the blog checks.

## Example invocation

> Run the handoff checklist on account `YOUR_ACCOUNT_ID`, public URL is `https://acmehvac.com`. They have a contact form and a blog.

## Tools used

- `mcp__brightsite__get_site_identity`
- `mcp__brightsite__list_pages` + `mcp__brightsite__get_page`
- `mcp__brightsite__list_error_pages`
- `mcp__brightsite__get_blog_settings`
- `mcp__brightsite__list_posts` + `mcp__brightsite__get_post`
- `mcp__brightsite__list_forms`
- `mcp__brightsite__list_redirects`
- `mcp__brightsite__get_analytics`
- `mcp__brightsite__get_global_code`

This skill is **read-only**. It never calls a write tool. The placeholder-text check in step 10 runs against content already loaded into context — it does not call `search_replace` or any other mutating tool.
