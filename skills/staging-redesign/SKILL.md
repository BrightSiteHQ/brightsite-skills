---
name: staging-redesign
description: Build a BrightSite redesign or large change safely on a staging site, preview it, then promote it live in one atomic flip — instead of editing the live site directly. Use when the user wants a "staging site," "redesign safely," "build a new version without breaking the live site," "preview before it goes live," "rebuild the homepage," "flip the whole site at once," "a safe place to work," or is doing a full visual rebuild (new fonts/layout/multiple pages) they don't want visitors to see until it's ready. For per-entity staged edits on the live site (a single page's unpublished draft), the normal `*_staged` + `publish_page` flow is enough — use this skill only when you want a full isolated copy.
---

# Build a Redesign on a Staging Site, Then Promote

A **staging site** is a full, independent, editable copy of the live site's content
(pages, layouts, components, forms, blog, media, global CSS/JS, Tailwind config, site
identity, redirects) that lives under the same account. You build the redesign on it
invisibly, preview it at a gated subdomain, then **promote** it — one atomic flip that
makes the staging content live and archives the old live site (restorable). Account-level
data (custom domain, analytics, form submissions, team, billing, tracking IDs) is shared
and never touched by a promote.

Use this instead of editing the live site directly whenever the change is big enough that
you don't want visitors seeing a half-finished state, or you need to flip several entities
(global code + a layout swap + N pages) at the same moment.

## When to use this vs. the normal staged-fields flow

| Situation | Use |
|---|---|
| One page's copy tweak, a new draft page, a blog post | Normal flow: `update_page` writes `*_staged`, `publish_page` ships it. Live stays as-is until you publish. |
| Full redesign, new global fonts/layout, multi-page rebuild, "flip it all at once", "let me preview the whole thing first" | **This skill** — a staging site. |

The staged-fields flow only stages *content* fields (`heex_staged`, `css_staged`, …).
Structural fields — `layout_id`, `slug`, `status`, `is_home_page`, `meta_*`, and anything
`update_site_identity` sets — take effect on the **live** site immediately, with no staged
variant. A staging site stages *everything*, which is why it's the right tool for a rebuild.

## Key mechanic: the `site` param

Every content tool (`create_page`, `update_page`, `update_layout`, `update_global_code`,
`create_component`, `request_upload`, `update_site_identity`, `list_pages`, `get_page`, …)
takes an optional **`site`** param:

- **omitted** or `"live"` → the live site (default, backwards-compatible).
- `"staging"` → the account's staging site.
- an explicit **site ID** (from `list_staging_sites`) → that specific site, as long as it's
  **active** (`live` or `staging`). Archived sites (rollback snapshots) are rejected —
  they're immutable history; `restore_staging_site` first if you need to edit one.

Authorization is still by `account_id` — you must be a member of the org, and the target
site must belong to it. Passing a staging site ID as `account_id` does **not** work (it's a
site, not an account); always pass `account_id` = the org and select the site with `site`.

## Workflow

### 1. Create the staging site

```
mcp__brightsite__create_staging_site { account_id: "<ORG_ID>" }
  → { id, name, slug, type: "staging", preview_url, ... }
```

This clones the live site synchronously (content + media). It is **plan-gated** — it fails
with a "plan" / "limit" message if staging isn't on the account's plan or a staging site
already exists. If one already exists, either reuse it or `discard_staging_site` first.

Keep the returned `id` and `preview_url`. You can also fetch them any time with
`mcp__brightsite__list_staging_sites` or `mcp__brightsite__get_staging_status`.

### 2. Author into staging — pass `site: "staging"` on every write

Build the redesign exactly as you normally would, but add `site: "staging"` (or the site
ID) to each call. The live site is untouched.

```
mcp__brightsite__update_global_code { account_id, site: "staging", css_staged: "...", tailwind_config_staged: "..." }
mcp__brightsite__update_layout      { account_id, site: "staging", id: "<layout_id>", heex_staged: "..." }
mcp__brightsite__create_page        { account_id, site: "staging", title, slug, heex, params_schema, ... }
mcp__brightsite__request_upload     { account_id, site: "staging", file_name, content_type }  # media lands on staging
```

If you're building editable content, load the **visual-editor-authoring** skill and apply
it — `site: "staging"` composes with everything there.

> Reads honor `site` too. `list_pages`/`get_page` with `site: "staging"` show the staging
> copy; without it, they show live. Use this to diff-by-eye or verify your writes landed.

### 3. Preview before promoting

The staging site is served at its own gated subdomain — the `preview_url` returned by
`create_staging_site` / `list_staging_sites`. Open it (or screenshot it) and confirm the
redesign looks right. Tracking (GA/GTM/Pixel) is suppressed on staging previews, so you
won't pollute analytics.

### 4. Review the diff

```
mcp__brightsite__staging_diff { account_id: "<ORG_ID>" }
  → { has_changes, totals: {added, modified, removed}, by_type, drift }
```

`drift` = live items that were edited *after* the staging copy was made — a promote would
overwrite them with the staging version. If `drift` is non-empty and you didn't intend it,
reconcile before promoting (re-apply that live edit on staging, or accept the overwrite).

### 5. Promote (atomic flip)

```
mcp__brightsite__promote_staging { account_id: "<ORG_ID>" }
```

Runs in the **background**: it validates the staging content and takes a full backup of the
current live site first, then flips `live_site_id`. It returns immediately with
`status: "promoting"`. **Poll `get_staging_status`** until the staging site is gone and the
live site is the former staging site.

The previously-live site is archived and restorable:

```
mcp__brightsite__restore_staging_site { account_id }                    # undo the last promotion
mcp__brightsite__restore_staging_site { account_id, site_id: "<archived_id>" }  # restore a specific archived site
```

### 6. (Optional) Discard instead of promoting

If the redesign is abandoned:

```
mcp__brightsite__discard_staging_site { account_id }   # permanently deletes staging content + media, frees the slot
```

## Anti-patterns to avoid

- **Passing the staging site ID as `account_id`.** It's a site, not an account —
  authorization is by org. Always pass `account_id` = the org and select the site with the
  `site` param. (`account_id: "<staging_site_id>"` returns "Access denied".)
- **Forgetting `site: "staging"` on a write during a redesign.** The call silently targets
  the LIVE site (that's the default). If a change you meant for staging shows up in
  `staging_diff` as live `drift`, you wrote to the wrong site — reverse it and redo with
  `site: "staging"`.
- **Using a staging site for a one-line copy fix.** Overkill. Use `update_page` +
  `publish_page` on the live site instead.
- **Promoting without previewing.** Always open the `preview_url` (or screenshot it) and
  run `staging_diff` first — a promote flips the public site.
- **Assuming promote is instant.** It's backgrounded (it backs up live first). Poll
  `get_staging_status` to confirm; don't report "done" off the immediate `"promoting"`.
- **Expecting analytics/form submissions/tracking IDs to move.** They're account-level and
  shared — a promote never touches them. Only site content moves.

## MCP tools used

- `mcp__brightsite__create_staging_site` — clone live → new staging site (plan-gated, sync).
- `mcp__brightsite__list_staging_sites` / `mcp__brightsite__get_staging_status` — find the
  staging site ID + `preview_url`; check whether one exists.
- `mcp__brightsite__staging_diff` — added/modified/removed counts + `drift` vs. live.
- `mcp__brightsite__promote_staging` — flip staging → live (background; poll to confirm).
- `mcp__brightsite__restore_staging_site` — undo a promotion / restore an archived site.
- `mcp__brightsite__discard_staging_site` — delete the staging site and its content.
- Every content tool (`create_page`, `update_page`, `update_layout`, `update_global_code`,
  `create_component`, `update_component`, `request_upload`, `update_site_identity`, …) with
  the optional `site` param to target `"staging"` (or a site ID) instead of live.
