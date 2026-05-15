---
name: bulk-seo-audit
description: Audit every page and post on a BrightSite site for on-page SEO issues. Checks title length, meta description length and presence, H1 presence and uniqueness, canonical URLs, og:image, robots directives, and slug quality. Outputs a prioritized fix list grouped by severity. Use when the user mentions "SEO audit," "on-page SEO," "check my pages for SEO," "find SEO issues," "before launch SEO check," or wants a pre-handoff QA pass on a client site.
---

# Bulk SEO Audit

Walk every page and post on a BrightSite site, evaluate on-page SEO basics, and produce a triaged report. This is for catching common issues at scale — it does NOT replace a real SEO audit that includes backlinks, Core Web Vitals, or rank tracking.

## When to use this

- Agency is about to hand off a client site and wants a final QA pass.
- Client's traffic has dropped and you want to rule out on-page issues.
- After a migration (e.g. after running the `migrate-wordpress-blog` skill).
- Periodic check on a live site every 30-90 days.

## Inputs you need from the user

1. **Account ID** — the BrightSite account to audit.
2. **Scope** — pages only, posts only, or both? Default: both.
3. **Severity threshold** — show all issues, or only critical + warnings? Default: all.

## Workflow

### Step 1: Pull the inventory

- `mcp__brightsite__list_pages` for all pages
- `mcp__brightsite__list_posts` for all posts

Note the counts and tell the user how many entities you're about to audit. If it's >200, ask for confirmation to proceed.

### Step 2: Fetch and evaluate each entity

For each page or post, call `mcp__brightsite__get_page` or `mcp__brightsite__get_post`. Pages return both `heex` (live) and `heex_staged` (latest unpublished); posts return `content` and `content_staged`. **Audit both** — issues in `*_staged` will go live the next time the user publishes. Group findings by which version is affected so the user knows whether to publish a fix or just edit the live entity.

Evaluate against these checks:

**Critical** (must fix — actively hurts SEO)
- `meta_title` empty or > 60 chars → truncated in SERPs
- `meta_description` empty → Google generates one, often badly
- `slug` contains spaces, capital letters, or special chars
- Two or more entities share the same slug
- HEEx/content body is empty or under 300 chars (check both live and staged)

**Warning** (should fix)
- `meta_title` < 30 chars (too short, missing keywords)
- `meta_description` < 70 chars or > 155 chars
- No H1 in the HEEx/content (check both live and staged)
- More than one H1 in the HEEx/content (check both live and staged)
- `canonical_url` set but points off-domain (suspicious — usually means duplicate content)
- `og_image_id` missing → poor social sharing previews. Note: most BrightSite templates fall back to the post's `feature_image_id` (or the site identity logo) when `og_image_id` is null, so only flag this if neither fallback exists. For pages, flag any page where `og_image_id` is null AND the site has no logo.
- Title and meta_title are identical (missed opportunity to differentiate for search vs. display)
- `meta_robots` contains `noindex` — could be intentional (a thank-you page, an internal tool page). Surface each match and ask the user to confirm whether it's deliberate.

**Info** (nice to fix)
- Slug is very long (> 60 chars)
- Slug contains stop words (`the`, `a`, `and`, `of`)
- `meta_description` doesn't mention any word from the title
- Post missing both `feature_image_url` and `feature_image_id`

### Step 3: Output the report

Format as a markdown table grouped by severity, then by entity type. For each issue, include:

- Severity marker (use `[!]` for critical, `[~]` for warning, `[i]` for info — no emojis unless user asked)
- Entity type and title
- The specific issue
- Suggested fix
- Page/post ID (so the user can jump to it in the editor)

End with a summary block:

```
Audit summary
-------------
Pages audited: 42
Posts audited: 118
Critical issues: 4
Warnings: 23
Info: 31

Top 3 fixes by impact:
1. 4 pages missing meta_description — generate descriptions for: [list]
2. 7 posts have meta_title > 60 chars — shorten: [list]
3. 2 pages share the slug "/services" — rename one
```

### Step 4: Offer to fix

After the report, ask the user:

> Want me to auto-fix the critical issues? I can generate meta descriptions and trim long titles. I'll show you each change before saving.

If they say yes, use `mcp__brightsite__update_page` / `mcp__brightsite__update_post` to apply fixes. Show each proposed change as a diff (`old → new`) and confirm before calling the update. Important: SEO metadata fields are not staged by the MCP; updating `meta_title`, `meta_description`, `meta_robots`, `canonical_url`, or `og_image_id` can affect live metadata immediately. Get explicit approval before each metadata update.

## Anti-patterns to avoid

- **Don't auto-fix without explicit approval per change for critical issues.** Auto-generating a meta description that's wrong about the page's content is worse than no description.
- **Don't flag stylistic preferences as issues.** "Title doesn't use power words" isn't an SEO issue, it's a copywriting opinion. Keep the audit objective.
- **Don't fetch all entities in parallel without throttling** on accounts with hundreds of pages. Sequential with a small batch size (e.g. 5 concurrent) is safer.
- **Don't trust `has_unpublished_changes: false` as proof of correctness.** Even published pages can have stale meta.

## Example invocation

> Run a full SEO audit on account `YOUR_ACCOUNT_ID`. Both pages and posts. Show me everything.

## Tools used

- `mcp__brightsite__list_pages`
- `mcp__brightsite__list_posts`
- `mcp__brightsite__get_page`
- `mcp__brightsite__get_post`
- `mcp__brightsite__update_page` / `mcp__brightsite__update_post` (only with explicit approval)
