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

## Tools used

- `mcp__brightsite__create_page` / `mcp__brightsite__update_page` — author pages; put
  `data-bs-edit` markers in `heex`, repeating content in `params_schema` collections.
- `mcp__brightsite__create_component` then `mcp__brightsite__update_component` — the
  two-call sequence to create an editable component (`props_schema` only on update).
- `mcp__brightsite__create_layout` / `mcp__brightsite__update_layout` — same markers apply
  to layout content.
- `mcp__brightsite__get_page` / `mcp__brightsite__get_component` — re-fetch to verify
  markers and a non-empty `props_schema` before reporting done.

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
