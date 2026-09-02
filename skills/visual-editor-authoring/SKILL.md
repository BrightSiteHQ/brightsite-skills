---
name: visual-editor-authoring
description: Author BrightSite pages, components, and layouts whose content is editable in the visual editor — so a non-technical user can click and change text, images, links, and component props without a second pass. Use when creating or editing a page/component/layout via the BrightSite MCP (create_page, update_page, create_component, create_layout), when building a site or template, or when the user says "make this editable," "the client should be able to edit this," "build the site so they can change it themselves," or "use the visual editor / components properly." Load this BEFORE writing HEEx, not after.
---

# Author Visual-Editor-Editable Content

BrightSite content authored as plain HEEx renders fine on the public site but is often
**not editable in the visual editor** — the user can't click an element to change its text,
swap an image, or edit a component's props. They then have to come back and ask for it to
be "made editable." This skill makes you author it correctly the first time.

Read **[editability-contract.md](editability-contract.md)** (in this skill's directory) in
full before authoring. It is the source of truth for every rule below. This file is the
how-to; the contract is the reference.

## When to use this

- You are about to call `create_page`, `update_page`, `create_component`,
  `update_component`, or `create_layout` and the content should be client-editable.
- You're building a whole site, a template, or a default/starter layout.
- The user wants non-technical end users (small-business owners) to self-edit content.

If you're only auditing or fixing an **existing** site for editability, use the
`visual-editor-audit` skill instead.

If you're doing a **full redesign or large rebuild** that shouldn't be visible on the live
site until it's ready, build it on a **staging site** — load the `staging-redesign` skill
and pass `site: "staging"` on the authoring calls below (it composes with every rule here).

## The rule that prevents 90% of second passes

> The editor sees exactly two editable things: elements tagged `data-bs-edit="field"`, and
> `component()` instances whose component has a **non-empty `props_schema`**. Raw HTML is
> invisible.

So before you write any HEEx, decide for each meaningful piece of content: is it a
`data-bs-edit` field, part of a component's `props_schema`, or a collection? If it's none
of those, the user can't edit it.

## Workflow

### Step 1: Read the contract

Read `editability-contract.md` in this directory. Internalize the decision rule and the
four silent traps.

### Step 2: Plan the editable surface before writing HEEx

For the page/component you're about to build, list the content the user will want to
change, and assign each a mechanism:

- **One-off scalar** (a heading, a single image, one button) → inline `data-bs-edit`.
- **A reused section** (hero, CTA band, footer block) → a `component()` with `props_schema`.
- **Repeating content** (pricing tiers, gallery, team, FAQ) → a collection (component
  `item_schema`, or inline `data-bs-collection`/`data-bs-item` **+ `data-bs-item-schema`**
  for the rich editor). Never a bare `:for`. Auto-number ordered items with `bs_index(idx)`.
  Put the `data-bs-collection`/`data-bs-item`/`data-bs-edit` markers on the rendered DOM too.
- **Shared design, per-page data** (related-card grids, page-specific lists) → a reusable
  component bound to a **page `@params` collection** (`%{cards: @params.related_cards}`),
  with the data + `item_schema` in the page's `params_schema`. **Never** pass the editable
  items as an inline literal in the `component(...)` call — the props panel can't read a
  literal, so the fields show empty.

### Step 3: Author with the markers

Apply the contract. The high-frequency rules:

- Add `data-bs-edit="field_name"` to every editable element.
- Use `data-bs-edit-type="richtext"` for any text with `<br>` or inline formatting — and
  inside richtext use inline `style="…"`, **never Tailwind classes** (they're dropped on
  save).
- Internal links: `href={page_url("page_id")}` **and** `data-bs-edit-type="page"`; for a
  CTA use `data-bs-edit-type="button"` (label + link edited as one grouped button).
- Inline collections with a known item shape: add `data-bs-item-schema='{…}'` to the
  `data-bs-collection` wrapper for the rich (collapsible / drag-reorder / typed) editor.
- **Any collection — including one rendered inside a component — needs the markers on its
  rendered DOM:** `data-bs-collection` on the container, `data-bs-item={idx}` per item,
  `data-bs-edit` per field. A populated `item_schema` alone does NOT give canvas
  hover/click-select; without the markers the panel shows empty fields and the cards don't
  highlight. Never a bare `:for` with `{item.field}` and no markers.
- Ordered lists: render the number with `bs_index(idx)` (auto-renumbers) — don't store it.
- Link/CTA props in reusable components: resolve the href through the small `resolve` helper
  (page ID → `page_url`, else raw) so the prop accepts a page ID *or* a literal
  path/anchor/tel. (See the contract's "smart `resolve` href helper.")
- Elixir `""` is **truthy**: guard `:if`/`||` on non-empty (`x && x != ""`), or a CTA with an
  empty label still renders / a fallback never fires.
- `<a>` with an icon/child markup: put `data-bs-edit-target` on the text child.

### Step 4: For components, do BOTH calls

`create_component` does **not** accept `props_schema`. A component without it has zero
editable fields. Always:

1. `create_component(...)` with the `heex`.
2. `update_component(..., props_schema: {…})` to define the editable props — each with a
   `label`, `type`, `default`, and an integer `order`.

A `type:"page"` prop's `default` must be a page **ID** (not a path).

### Step 5: Verify before reporting done

Run the pre-ship checklist from the contract. Concretely, the page must have ≥1 editable
node, every reused section must be a component with a non-empty `props_schema`, every
repeating block must be a collection, and internal links must use `page_url` +
`data-bs-edit-type="page"`. If you can, re-fetch with `get_page` / `get_component` and
confirm the markers are present in the stored HEEx and the `props_schema` is non-empty.

## Anti-patterns to avoid

- **Shipping a page of clean `<div>`s with no `data-bs-edit` and no components.** It looks
  done and edits nothing. This is the #1 cause of the second pass.
- **Creating a component and stopping** — leaving `props_schema` at `{}`. The editor shows
  "no editable properties." Always follow `create_component` with `update_component`.
- **Tailwind classes inside a richtext field.** `class="text-teal-500 italic"` is silently
  dropped on save. Use `style="color:#14b8a6; font-style:italic;"`.
- **`type:"text"` on text containing `<br>` or `<span>`.** The first edit flattens it to
  plain text. Use `richtext`.
- **Bare `:for` loops for editable repeating content.** One opaque block — no add/remove/
  reorder. Use a collection.
- **A component collection with `item_schema` but no markers on the rendered loop.** The
  panel shows empty placeholder fields and the canvas cards don't hover/select. Add
  `data-bs-collection`/`data-bs-item`/`data-bs-edit` to the rendered DOM — the `item_schema`
  is necessary but not sufficient.
- **Passing editable items as an inline literal** in `component("x", %{cards: [...]})`. The
  panel reads the instance, not the literal → empty fields. Bind to `@params.<collection>`.
- **Hardcoded `href="/slug"` for internal links.** No page picker, breaks on slug change.
  Use `page_url(...)` + `data-bs-edit-type="page"`.
- **Trusting Elixir truthiness with empty strings.** `"" || @fallback` is `""`, and
  `:if={@label}` is true when `@label == ""`. Guard on `x && x != ""`.
- **Props with no `order` key.** They list in random order; users notice. Set `order` on
  every prop.

## The dangerous step

The component two-call sequence (Step 4). It is the easiest thing to get wrong because
`create_component` succeeds and looks complete — but a component with an empty
`props_schema` is locked. Never report a component done until `update_component` has set a
non-empty `props_schema`.

## Builtin navigation, footer & cookie consent

Every site has three **builtin components** — the navigation (menu), the footer, and the
cookie consent banner — marked with `builtin_kind` (`"navigation"` / `"footer"` /
`"cookie_consent"`) in `list_components`. Rules:

- **Never hand-build a header or footer** as an ordinary component or inline layout HEEx.
  Layouts render those builtins with `builtin("navigation")` and `builtin("footer")` (kind
  reference — survives renames; `component("slug")` also works but is fragile).
- The **cookie consent banner needs NO layout reference**: when its `enabled` value is
  true it is injected automatically into every rendered page. **Never write
  `builtin("cookie_consent")`** (or `component("cookie-consent")`) into a layout — that
  double-renders the banner. Turning it on is a single `update_builtin_content` call with
  `{"enabled": true}`.
- Builtins **cannot be deleted**, and their `name`, `slug`, and `props_schema` are
  system-managed: `update_component` returns `builtin_locked_fields` if you try to change
  them. Passing them back unchanged (get → update round-trip) is fine.
- Their **design** (`heex_staged`/`css_staged`/`js_staged`) IS editable and publishes like
  any component — but keep referencing the existing props (`@items`, `@buttons`,
  `@layout`, `@columns`, `@social_style` for nav/footer; `@enabled`, `@message`,
  `@accept_label`, `@show_decline`, `@decline_label`, `@policy_label`, `@policy_page`,
  `@policy_url`, `@position`, `@theme` for the banner) or the user's settings stop
  affecting the design (the editors flag such dead settings). The banner's consent script
  lives in its `js` field — keep the `data-bs-consent="accepted|declined"` buttons and the
  `#bs-cookie-consent` wrapper id when restyling, or consent stops being recorded.
- Their **content** (menu items, buttons, footer link columns, banner text and toggles) is
  edited via `get_builtin_content` / `update_builtin_content` (kind `"navigation"`,
  `"footer"`, or `"cookie_consent"`) — NOT by writing props_schema. `get_builtin_content`
  returns the current values plus the authoritative props_schema (valid keys and select
  options); `update_builtin_content` merges a values JSON object and applies LIVE
  immediately (use a staging site to stage menu changes). Users edit the same values in
  the app at /app/navigation, /app/footer, and /app/cookie-consent.
- Duplicating a builtin yields an ordinary, unprotected component.

### Navigation values

`items` (menu entries), `buttons` (list of `{label, page|url, new_tab, style: solid|outline}`),
`layout` (`links-right` | `links-center` | `links-left` | `stacked`), `show_logo` (bool),
`dropdown_style` (`dropdown` | `mega`), `sticky` (bool).

**Menu nesting — three levels, no more.** A top-level item's `children` are its dropdown
entries. Each dropdown entry is one of two kinds:

- a plain **link** (the default, or `type: "link"`) — `{label, page|url, new_tab}`
- a **section** (`type: "section"`) — a heading whose own `children` are that section's
  links. Sections are how a mega menu gets its labelled columns. A section is not
  clickable, so give it no page/url, and it is the last level: its links take no children.
  A section must have at least one link.

A section may also carry `image` (a media file ID) — an optional picture rendered **above**
its heading, for image-led menu columns. Omit it (or `""`) for a plain text column.

**Mega-menu panel** — optional, on TOP-LEVEL items only, all default `""` and render
nothing when empty: `promo_image` (media file ID), `promo_label` (e.g. "MOST POPULAR"),
`promo_link_label` (e.g. "VIEW ALL SERVICES"), `promo_page` **or** `promo_url`.
Set `dropdown_style` to `"mega"` for the full-width panel these are designed for.

### Footer values

`columns` (list of `{title, links: [{label, page|url, new_tab}]}`), `layout`
(`columns` | `brand` | `centered` | `minimal`), `show_logo` (bool), `copyright` (text,
`""` = automatic), `social_style` (`icon` | `icon_text` | `text` | `hidden`). Footer social
ACCOUNTS come from site identity (`update_site_identity` `social_profiles`), not from here.

### Cookie consent values

`enabled` (bool), `message` (text), `accept_label`, `show_decline` (bool), `decline_label`,
`policy_label`, `policy_page` (a page ID) **or** `policy_url`, `position`
(`bottom` | `bottom-left` | `bottom-right`), `theme` (`dark` | `light`).

Consent is **opt-out**: the site's third-party trackers (GA4/GTM/Meta Pixel) run until a
visitor declines, then stop from their next page load. BrightSite's own analytics is
cookieless and unaffected either way, so it keeps working regardless of the banner.

## Site-wide template helpers

Available in any tenant template (pages, layouts, components):

- `@site` — site identity: `name`/`title`, `tagline`, `logo_url`, `phone`, `email`,
  `address_street`, `address_city`, `address_state`, `address_zip`, `address_country`,
  `social_profiles` (list of URLs). Prefer these over hardcoding contact details, so
  editing Site Identity updates every page.
- `social_icon(url)` — brand SVG for a social profile URL (raw, safe to interpolate).
- `social_name(url)` — the platform's display name for that URL (e.g. "Instagram").
  Both are backed by the same platform list Site Identity writes, so pair them with
  `@site[:social_profiles]` rather than hand-rolling a URL-to-icon `case`.

## Tools used

- `mcp__brightsite__create_page` / `mcp__brightsite__update_page` — author pages; put
  `data-bs-edit` markers in `heex`, repeating content in `params_schema` collections.
- `mcp__brightsite__create_component` then `mcp__brightsite__update_component` — the
  two-call sequence to create an editable component (`props_schema` only on update).
- `mcp__brightsite__create_layout` / `mcp__brightsite__update_layout` — same markers apply
  to layout content.
- `mcp__brightsite__get_page` / `mcp__brightsite__get_component` — re-fetch to verify
  markers and a non-empty `props_schema` before reporting done.
- `mcp__brightsite__get_builtin_content` / `mcp__brightsite__update_builtin_content` — read
  and write builtin CONTENT (kind `navigation` | `footer` | `cookie_consent`). Always get
  before update; the response carries the valid keys and select options.

## Uploading images to the media library

To reference a real image in a page (`media(file_id)` / `media(file_id, aspect: "16:9")`
/ `media_url(file_id)`), the file must first exist in the org's media library. Use the
**two-step presigned flow** — do NOT use `mcp__brightsite__upload_file` (the local-path
convenience tool returns a bare `Internal error`):

1. `mcp__brightsite__request_upload` `{account_id, file_name, content_type}` → returns
   `{file_id, upload_url}` (a presigned Cloudflare R2 URL, ~2h expiry).
2. HTTP **PUT** the raw bytes to `upload_url` with a matching `Content-Type` header:
   `curl -X PUT -H "Content-Type: image/jpeg" --data-binary @file.jpg "$URL"` → expect **200**.
3. `mcp__brightsite__complete_upload` `{account_id, file_id, file_name, name,
   content_type, width, height, size}` → creates the DB media record and returns
   `{id, thumb_url, md_url, lg_url, orig_url}`. The returned `id` == the `file_id` you
   passed; use it in `media(...)`.

Gotchas: use a `.jpg` extension, never `.jpeg` (the CDN signed-URL pipeline 404s on
`.jpeg` objects). Resize huge originals (3500px+) down to ~1400–2000px before the PUT so
uploads stay fast. For blog feature images, the same `id` is what you pass as
`feature_image_id` (preferred over `feature_image_url`, which is for external URLs).

## Images in email (newsletters, transactional templates)

CDN image URLs negotiate their format on the `Accept` header: browsers get WebP, other
clients get the source format. Gmail's image proxy advertises WebP support and then
transcodes it to JPEG, flattening transparency onto black — a white-on-transparent logo
becomes a black box. So an email must never receive a negotiated URL.

- From `mcp__brightsite__list_media` / `complete_upload`, use the **`email_url`** field:
  the `lg` size with the format pinned inside the signed path (PNG, or JPEG when the
  source is a JPEG). Never WebP, alpha preserved. `md_url` / `lg_url` are for web pages.
- In HEEx, pin explicitly: `media(file_id, format: "png")` or
  `media_url(file_id, format: "png")`. Accepted formats: `png`, `jpeg`, `webp`, `gif`,
  `avif`. An unknown value renders a visible `media-error` span rather than an image.
- Photos can stay JPEG; only assets with transparency need PNG.
