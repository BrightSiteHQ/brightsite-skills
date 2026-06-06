---
name: visual-editor-audit
description: Audit an existing BrightSite site for visual-editor editability — walk every page, component, and layout and flag what a non-technical user cannot edit (no data-bs-edit markers, components with empty props_schema, bare loops for repeating content, hardcoded internal links), then optionally fix it. Use when the user says "check what's editable," "the client can't edit X in the editor," "why can't I edit this," "audit the site for the visual editor," "make the existing site editable," "fix editability," or wants a pre-handoff QA pass on whether the site is editable. For authoring new content correctly the first time, use the visual-editor-authoring skill instead.
---

# Visual Editor Editability Audit

Walk every page, component, and layout on a BrightSite site and report what is **not
editable in the visual editor** — the things a non-technical client can't click and change
— then optionally fix them. This is the reactive counterpart to `visual-editor-authoring`
(which prevents the problem at create time).

The rules this audit enforces live in
**[../visual-editor-authoring/editability-contract.md](../visual-editor-authoring/editability-contract.md)**.
Read that file first; it is the source of truth for every check below.

## When to use this

- A client says they can't edit something in the visual editor.
- Pre-handoff QA: confirm the site is actually self-editable before giving it to the client.
- After a site was built quickly (or by another tool) and you suspect it's full of raw HTML.
- After running `visual-editor-authoring` on a large build, as a verification pass.

If you're creating new content, don't audit after — author it right the first time with the
`visual-editor-authoring` skill.

## Inputs you need from the user

1. **Account ID** — the BrightSite account to audit.
2. **Scope** — pages, components, layouts, or all. Default: all.
3. **Fix mode** — report only, or report then offer to fix? Default: report, then offer.

## The contract in one sentence

> The editor sees exactly two editable things: elements tagged `data-bs-edit="field"`, and
> `component()` instances whose component has a **non-empty `props_schema`**. Everything
> else is invisible.

Every check below is a way the authored content fails that rule.

## Workflow

### Step 1: Pull the inventory

- `mcp__brightsite__list_pages`
- `mcp__brightsite__list_components`
- `mcp__brightsite__list_layouts`

Report the counts before you start. If it's a large site (>150 entities), audit in batches
(≤5 concurrent) and tell the user you're throttling.

### Step 2: Fetch and evaluate each entity

For each entity, fetch with `get_page` / `get_component` / `get_layout`. Pages and layouts
return both live (`heex`) and staged (`heex_staged`) bodies — **audit both**, because a
staged body goes live on the next publish. Components return their `heex` and `props_schema`.

Evaluate against these checks:

**Critical** (the user genuinely cannot edit this content)

- **Page/layout has zero editable nodes** — no `data-bs-edit` anywhere AND no embedded
  `component(...)`. The whole body is one opaque "Page Content" block.
- **Component has an empty `props_schema` (`{}`)** but its `heex` contains content/props.
  The editor shows "no editable properties." (Caused by `create_component` without the
  follow-up `update_component`.)
- **Repeating content rendered with a bare `:for`** (a `:for` with no surrounding
  `data-bs-collection`/`data-bs-item` and not driven by a component `item_schema`). Items
  can't be added, removed, reordered, or individually edited.
- **A collection prop's rendered loop has no `data-bs-collection`/`data-bs-item`/
  `data-bs-edit` markers** — even when the component's `props_schema` has a populated
  `item_schema`. Symptom: the panel shows empty placeholder fields for each item, and the
  cards on the canvas don't hover-highlight or click-select. The `item_schema` makes the prop
  appear; the rendered markers make it editable. Fix: add the three markers to the loop in the
  component HEEx (`data-bs-collection="<prop>"`, `data-bs-item={idx}`, `data-bs-edit="<field>"`).
- **An editable prop passed as an inline literal** in a `component(...)` call — e.g.
  `component("related", %{cards: [%{title: …, href: "/team"}]})`. The props panel reads the
  component instance, not the literal, so those fields show as **empty defaults** and the
  user can't edit the real per-page values. Fix: move the data into the page's `params_schema`
  as a collection and bind it (`%{cards: @params.related_cards}`).

**Warning** (editable, but the editing experience is broken or fragile)

- **Internal links hardcoded** as `href="/slug"` instead of `href={page_url("id")}` — no
  page picker, breaks on slug change. Flag missing `data-bs-edit-type="page"` on internal
  `<a>` tags too.
- **`data-bs-edit-type="text"` on an element containing `<br>` or child tags** — the first
  edit will flatten it. Should be `richtext`.
- **Tailwind classes inside a richtext field** (`class="…"` on a `<span>`/child within a
  `data-bs-edit-type="richtext"` element) — dropped on save; should be inline `style="…"`.
- **`<a>` with an icon/child markup but no `data-bs-edit-target`** on the text child —
  editing the label destroys the icon.
- **Component props with no `order` key** — fields list in arbitrary order in the panel.
- **`type:"page"` component prop whose default is a path** (`/contact`) not a page ID — the
  picker shows "— Select a page —".
- **A reusable component takes a page-link/CTA prop but renders `href={page_url(@x)}`
  directly** (no `resolve` helper) — breaks if the prop holds a literal path/anchor/`tel:`
  instead of a page ID. Suggest the smart `resolve` helper (page ID → `page_url`, else raw).
- **`:if` guard or `||` fallback that relies on Elixir truthiness with possibly-empty
  strings** (`:if={@label}`, `"" || @fallback`) — `""` is truthy, so an empty-label CTA still
  renders and the fallback never fires. Should test `@label && @label != ""`.

**Info** (nice to improve)

- Long page with many `data-bs-edit` fields but no `data-bs-section` grouping — the element
  tree is a flat wall; suggest wrapping logical sections in `data-bs-section="Label"`.
- A section's markup is duplicated across multiple pages as raw HTML — candidate to extract
  into a reusable component.
- **Inline `data-bs-collection` without `data-bs-item-schema`** — it works, but only gets the
  basic one-item-at-a-time editor. If the items have a clear shape, suggest adding
  `data-bs-item-schema='{…}'` for the rich (collapsible / drag-reorder / typed) editor.
- **Ordered-list numbers stored as an editable field** — an item field like `num`/`number`
  ("01", "02") that's really just the position. Suggest rendering it with `bs_index(idx)` (or
  `bs_index(idx, pad: 2)`) and dropping it from the data, so it auto-renumbers on reorder.
- **A CTA authored as two separate fields** (a `text` field + a `page` link side by side, or
  a `type:"page"` link whose label is a separate prop) — suggest a single
  `data-bs-edit-type="button"` (page) or a `type:"button"` component prop so label + link
  edit as one grouped button.

### Step 3: Output the report

Markdown table grouped by severity, then by entity type. For each finding include:

- Severity marker — `[!]` critical, `[~]` warning, `[i]` info (no emoji unless asked).
- Entity type + title, and whether the issue is in the live or staged body.
- The specific defect and the contract rule it violates.
- The concrete fix (the marker/attribute to add, or the two-call sequence to run).
- The entity ID, so the user can jump to it.

End with a summary block:

```
Editability audit
-----------------
Pages audited:      14   (3 not editable)
Components audited:  9   (2 with empty props_schema)
Layouts audited:     2

Critical: 5
Warnings: 11
Info: 4

Top fixes by impact:
1. 3 pages have zero editable nodes — add data-bs-edit / componentize: [list]
2. 2 components have empty props_schema — run update_component: [list]
3. Gallery page uses a bare :for — convert to a data-bs-collection: [id]
```

### Step 4: Offer to fix

Ask before changing anything:

> Want me to fix the critical issues? I'll add `data-bs-edit` markers, set `props_schema`
> on the empty components, and convert bare loops to editable collections. I'll show each
> change as a diff and confirm before saving.

If yes, apply fixes with `update_page` / `update_component` / `update_layout`, following the
authoring contract. Show each change as a diff (`old → new`) and get approval per entity.
Key reminders while fixing:

- Setting a component's `props_schema` is an `update_component` call — `create_component`
  can't do it.
- Content edits to pages/layouts go to the **staged** body; tell the user they must publish
  (`publish_page` / `publish_layout`) to make the now-editable version live.
- When you add `data-bs-edit` markers, preserve the existing rendered output — markers
  change editability, not appearance.

## Anti-patterns to avoid

- **Don't auto-fix without per-entity approval.** Rewriting a page's HEEx to add markers
  can subtly change rendering if done carelessly. Show the diff.
- **Don't only audit the live body.** A clean live page with a non-editable staged body
  ships the problem on the next publish. Check both.
- **Don't flag styled-but-static decorative elements as "not editable."** A background
  flourish the client never needs to edit isn't a defect. Focus on content: headings,
  body copy, images, links, CTAs, repeating items.
- **Don't report a component "fixed" after `create_component`-style edits alone** — verify
  the `props_schema` is non-empty after the `update_component`.
- **Don't fetch hundreds of entities in parallel.** Throttle to ~5 concurrent.

## Example invocation

> Audit account `YOUR_ACCOUNT_ID` for visual-editor editability — pages, components, and
> layouts. Tell me what the client can't edit, then fix the critical stuff.

## Tools used

- `mcp__brightsite__list_pages` / `mcp__brightsite__list_components` / `mcp__brightsite__list_layouts`
- `mcp__brightsite__get_page` / `mcp__brightsite__get_component` / `mcp__brightsite__get_layout`
- `mcp__brightsite__update_page` / `mcp__brightsite__update_component` / `mcp__brightsite__update_layout` (only with explicit per-entity approval)
- `mcp__brightsite__publish_page` / `mcp__brightsite__publish_layout` (to make staged fixes live, with approval)
