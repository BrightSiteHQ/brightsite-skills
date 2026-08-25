# The BrightSite Editability Contract

This is the shared reference for what makes BrightSite content editable in the visual
editor. Both the `visual-editor-authoring` and `visual-editor-audit` skills read this
file. It is the source of truth — if a rule isn't here, don't assume it.

The visual editor lets a non-technical user click an element on the live preview and
change its text, image, link, or component props without touching code. That only works
if the authored HEEx carries the right markers. If it doesn't, the element is **invisible
to the editor** — it renders fine on the public site, but the user cannot select or edit
it, and they will have to come back and ask for it to be "made editable."

## The one-sentence mental model

> **The editor sees exactly two kinds of editable nodes:**
> 1. inline elements you tag with `data-bs-edit="field_name"`, and
> 2. `component()` instances whose component has a **non-empty `props_schema`**.
>
> **Everything else — no matter how clean the HTML — is invisible.**

A page made of beautiful, semantic, perfectly-styled `<div>`s with zero `data-bs-edit`
attributes and zero components is, from the editor's point of view, one opaque block of
"Page Content" the user cannot edit. This is the single most common reason a site needs a
second pass.

## What is auto-injected vs. what you must author

The renderer injects some wrappers for you at render time. **It never injects the field
markers — those are your job.**

| Marker | Who writes it | Notes |
|---|---|---|
| `data-component` / `data-component-index` | **Auto** (render time) | Added when you embed `component("slug")`. You never write these. |
| `data-bs-scope="page"` | **Auto** (render time) | Wraps page body so the editor can scope page-level fields. |
| `data-bs-edit` | **You** | The field marker. No editor visibility without it (or a component). |
| `data-bs-edit-type` | **You** (when needed) | Overrides the inferred type. |
| `data-bs-edit-target` | **You** (when needed) | Scopes which child text is edited (see links). |
| `data-bs-collection` / `data-bs-item` | **You** | For inline repeating content. |
| `data-bs-section` | **You** (optional) | Display-only grouping in the element tree. |
| `data-bs-order` | **You** (optional) | Display-only ordering within a section (tree + panel). |
| `props_schema` on a component | **You**, via `update_component` | `create_component` does NOT accept it — see the two-call rule below. |

## Mechanism 1 — `data-bs-edit` scalar fields

Tag any element with `data-bs-edit="field_name"`. The value is the field's storage key.
The editor finds editable elements with `querySelectorAll('[data-bs-edit]')`.

```heex
<%!-- NOT EDITABLE: invisible to the editor --%>
<h1 class="text-4xl font-bold">Root-cause care for modern medicine</h1>

<%!-- EDITABLE: appears in the sidebar as a text field --%>
<h1 class="text-4xl font-bold" data-bs-edit="hero_headline">Root-cause care for modern medicine</h1>
```

### Edit types

The type decides the editing UI. It's **inferred from the tag** when `data-bs-edit-type`
is absent:

- `<img>` → `image`
- `<a>` → `link`
- everything else → `text`

Override with `data-bs-edit-type`. The **only** values the code honors are:

| `data-bs-edit-type` | Editing UI | Use on |
|---|---|---|
| `text` (default) | plain-text field, edits `textContent` | headings/paragraphs with NO child tags |
| `richtext` | rich editor, edits `innerHTML` | text with `<br>`, `<em>`, `<strong>`, `<u>`, `<a>`, styled `<span>` |
| `link` | text + URL fields | `<a>` to an external/arbitrary URL |
| `page` | text + page-dropdown picker | `<a>` to an internal page |
| `button` | text + page-dropdown picker (grouped) | a CTA `<a>` you want edited as one "button" (label + link together) |
| `image` / `media` | media picker | `<img>` |
| `color` | color picker | any element; **never inferred — must be explicit** |

### text vs. richtext (a real trap)

`text` reads `el.textContent` — it **strips all child tags**. If you put
`data-bs-edit="x"` (type text) on an element containing `<br>` or `<span>`, the user's
first edit flattens it to plain text and the markup is gone.

Use `richtext` whenever the field contains line breaks or inline formatting:

```heex
<%!-- WRONG: type text strips the <span> on first edit --%>
<h1 data-bs-edit="headline">Hello <span style="color:#cda19b">World</span></h1>

<%!-- CORRECT: richtext preserves inline markup --%>
<h1 data-bs-edit="headline" data-bs-edit-type="richtext">Hello <span style="color:#cda19b">World</span></h1>
```

### THE RICHTEXT + TAILWIND TRAP (load-bearing)

Inside a `richtext` field, **Tailwind classes are silently dropped on save.** The richtext
editor only round-trips inline styles (and converts `text-[#hex]` color classes to inline
`style`). A `class="text-gold italic"` becomes nothing after the first save.

```heex
<%!-- WRONG inside richtext: classes vanish on save --%>
<h1 data-bs-edit="headline" data-bs-edit-type="richtext">
  We make it <span class="text-teal-500 italic">simple</span>.
</h1>

<%!-- CORRECT: inline styles persist --%>
<h1 data-bs-edit="headline" data-bs-edit-type="richtext">
  We make it <span style="color:#14b8a6; font-style:italic;">simple</span>.
</h1>
```

### Links with child markup — `data-bs-edit-target`

Editing a `link`/`page` field replaces the element's text. If the `<a>` contains an icon
or other child markup, that gets destroyed. Mark the text child with `data-bs-edit-target`
so only that text is replaced:

```heex
<%!-- WRONG: editing the label nukes the <svg> icon --%>
<a data-bs-edit="cta" data-bs-edit-type="page" href={page_url("contact_id")}>
  Book a call <svg>…</svg>
</a>

<%!-- CORRECT: only the targeted span's text is edited; icon survives --%>
<a data-bs-edit="cta" data-bs-edit-type="page" href={page_url("contact_id")}>
  <span data-bs-edit-target>Book a call</span> <svg>…</svg>
</a>
```

### Internal links — reference by page ID, render with `page_url(...)`

**RULE: for any link to a page ON THIS SITE, reference it by its page ID via
`page_url("id")` — never a hardcoded path. Use a literal path/URL ONLY for external links**
(`https://…`) or non-page targets (`#anchor`, `tel:`, `mailto:`). Referencing internal pages
by ID makes them slug-safe (the link survives a URL change) and gives the editor a page picker.

```heex
<%!-- WRONG: hardcoded path to an internal page — raw URL field, breaks on slug change --%>
<a data-bs-edit="cta" href="/contact">Contact us</a>

<%!-- CORRECT: internal page referenced by ID, slug-safe, gives a page picker --%>
<a data-bs-edit="cta" data-bs-edit-type="page" href={page_url("contact_page_id")}>Contact us</a>

<%!-- CORRECT: external link / non-page target uses a literal URL --%>
<a data-bs-edit="cta" data-bs-edit-type="link" href="https://example.com">External</a>
<a href="tel:+12077802000">Call us</a>
```

> **Internal pages are referenced by page ID EVERYWHERE — no exceptions.** This now holds for
> all three mechanisms: page-level inline fields, component props, AND inline-collection `page`
> sub-fields. There is no longer a path-based carve-out. Store the page **ID** and render with
> `page_url(...)`. (Back-compat: inline-collection values authored as paths before this change
> still resolve — `page_url` returns a literal path/anchor/`tel:`/`mailto:`/`http` untouched, and
> the editor's page-picker highlights a page whose **id OR path** matches the stored value. So old
> sites keep working, and re-saving an item migrates it to the ID. New authoring should always use
> the ID.)

> **Path vs ID — one rule now (get this wrong and the page-picker shows "— Select a page —").**
> Every page reference stores/picks a page **ID** and renders through `page_url(...)`. The only
> wrinkle left is *how the value reaches the DOM*, and it no longer changes what you author:
>
> | Mechanism | Author it as |
> |---|---|
> | Page-level inline field `data-bs-edit-type="page"` / `"button"` | `href={page_url("ID")}` (or the `resolve` helper). A literal `href="/path"` still works for one-off external/non-page targets, but internal pages use the ID. |
> | **Inline-collection `page` sub-field** (`item_schema` `{"type":"page"}`) | store an **ID** in the default; render `href={page_url(item.field)}` — same as component arrays. (`page_url` passes a literal path through untouched, so legacy path defaults still render.) |
> | Component prop `type:"page"` (in `props_schema`) | store an **ID**; render `href={page_url(@prop)}` (or the `resolve` helper). |
>
> Bottom line: **internal page → page ID → `page_url`, in all three places.** A literal
> path/anchor/`tel:`/`mailto:`/`http` is only for genuinely external or non-page targets, and
> `page_url`/`resolve` pass those through unchanged so you never have to special-case them.

#### The smart `resolve` href helper (for components that take a page-link prop)

A component prop of `type:"page"` stores a page **ID**, which must be turned into a URL with
`page_url(id)` at render. But authors (and inline defaults) sometimes pass a raw path
(`/contact`), an anchor (`#tiers`), a `tel:`/`mailto:`, or a full `http` URL — and calling
`page_url("/contact")` on a path mangles it. Wrap the value in a small resolver that only
applies `page_url` to a **bare page ID**, leaving everything else untouched. This makes the
prop accept both an editor-picked page ID *and* a hand-written path/anchor/tel without
breaking either:

```heex
<% resolve = fn v -> if is_binary(v) and v != "" and not String.starts_with?(v, ["/", "#", "tel:", "mailto:", "http"]), do: page_url(v), else: v end %>
<a href={resolve.(@cta_link)} data-bs-edit="cta_link" data-bs-edit-type="button">{@cta_label}</a>
```

Use this in any reusable component whose link/CTA props might receive either a page ID or a
literal href (heroes, CTA bands, related-card grids).

### Buttons — `data-bs-edit-type="button"`

A CTA is a label **and** a link that belong together. `data-bs-edit-type="button"` on an
`<a>` gives the user one grouped editor (a "Text" field + a "Links to" page picker) instead
of two separate fields. It behaves like `page` (stores a path, slug-safe) but reads as a
single button in the panel — prefer it for CTAs over a bare `page` link.

```heex
<a data-bs-edit="cta_primary" data-bs-edit-type="button" href={page_url("contact_id")} class="btn-accent">Contact Us</a>
```

(The component-prop analog is a `type:"button"` prop with `label`+`link` sub-fields — see
the `props_schema` section.)

## Mechanism 2 — Components with `props_schema`

A reusable section (hero, CTA band, feature grid, pricing table) should be a **component**.
Once embedded with `component("slug")`, the renderer auto-wraps it in `data-component`, and
**every prop defined in the component's `props_schema` becomes editable** — grouped under
one expandable node in the element tree. This is cleaner than scattering `data-bs-edit`
across inline HTML and is the only way to make a section reusable across pages.

### THE TWO-CALL RULE (the #1 component trap)

**`create_component` does NOT accept `props_schema`.** Its only params are `account_id`,
`name`, `description`, `heex`, `css`, `js`. A component created and left there has
`props_schema = {}` and surfaces **zero** editable fields — the editor shows "This
component has no editable properties." You must follow up with `update_component` to set
the schema:

```
1. create_component(account_id, name: "Page Hero", heex: "…")   → component with props_schema {}
2. update_component(account_id, id: <id>, props_schema: { … })  → NOW its props are editable
```

If you stop after step 1, you have shipped a locked component. Always do both.

### props_schema shape

```json
{
  "eyebrow":   {"type": "text",  "label": "Eyebrow",  "default": "The Approach", "order": 1},
  "headline":  {"type": "richtext", "label": "Headline", "default": "Root-cause care", "order": 2},
  "image":     {"type": "media", "label": "Image",    "default": "", "order": 3},
  "cta_label": {"type": "text",  "label": "Button text", "default": "Book a call", "order": 4},
  "cta_page":  {"type": "page",  "label": "Button link", "default": "<page_id>", "order": 5}
}
```

- Each key is a prop name → available as `@prop_name` (or `@props.name`) in the HEEx.
- `type` values match the edit types above (`text`, `richtext`, `media`/`image`, `page`,
  `color`) plus `array`/`collection` for repeating props (next section) and `button`/`object`
  for a grouped prop (below).
- **`order`** (integer) controls the field order in the panel. Without it, fields default
  to `order 999` and list in arbitrary map order — users notice. Always set `order`.
- A `type:"page"` prop's `default` must be a page **ID**, not a path.

### A `button`/`object` prop (one prop, grouped sub-fields)

For a CTA, use ONE `type:"button"` prop with `label` + `link` sub-fields rather than two
separate text + page props — the editor shows them as a single grouped "button," and the
value resolves to a map. (`object` is the generic form; `button` is the same thing tuned for
a CTA.) The sub-fields live under `fields` — the object analog of a collection's
`item_schema`.

```json
{
  "cta": {
    "type": "button", "label": "Primary CTA", "order": 5,
    "fields": {
      "label": {"type": "text", "label": "Text",     "default": "Contact Us"},
      "link":  {"type": "page", "label": "Links to",  "default": "<page_id>"}
    }
  }
}
```

```heex
<%!-- value is a map; reference sub-fields --%>
<a href={page_url(cta.link)} data-bs-edit="cta" data-bs-edit-type="button">{cta.label}</a>
```

### Component HEEx consumes the props

```heex
<%!-- the component's heex body --%>
<section class="hero">
  <p>{@eyebrow}</p>
  <h1>{@headline}</h1>
  {media(@image, alt: "")}
  <a href={page_url(@cta_page)}>{@cta_label}</a>
</section>
```

```heex
<%!-- the page embeds it — one editable, grouped node; reusable across pages --%>
{component("page-hero", %{eyebrow: "The Approach", headline: "Root-cause care", cta_page: "contact_id"})}
```

## Mechanism 3 — Repeating content (collections)

For anything that repeats — pricing tiers, gallery images, team members, FAQ, testimonials,
table rows — a bare `:for` loop is **one opaque block**; the user can't add, remove, reorder,
or edit individual items. There are two ways to make repeating content editable.

### Option A (preferred for reuse): a component prop of type `array`/`collection`

Give the component a prop whose schema has an **`item_schema`**. That produces a full
add / remove / reorder panel. **Both `type:"array"` and `type:"collection"` work** — they
render the identical panel; the requirement is the `item_schema`, not the type literal.

```json
{
  "tiers": {
    "type": "collection",
    "label": "Pricing tiers",
    "item_schema": {
      "name":  {"type": "text", "label": "Name",  "order": 1},
      "price": {"type": "text", "label": "Price", "order": 2}
    },
    "default": [{"name": "Launch", "price": "$39"}, {"name": "Grow", "price": "$79"}]
  }
}
```

> **CRITICAL — the component's rendered DOM still needs the collection markers.** A
> populated `item_schema` makes the prop appear in the panel, but the **canvas↔panel sync**
> (hover-highlight, click-to-select a card on the live preview, and per-item field editing)
> only works if the rendered loop carries `data-bs-collection` on the container,
> `data-bs-item={idx}` on each item, and `data-bs-edit` on each field. A bare `:for` with
> `{tier.name}` and no markers gives you a panel that shows **empty placeholder fields and no
> hover/select on the canvas** — the single most common "the collection looks broken in the
> editor" bug. The renderer auto-injects `data-component`, but it does **not** inject these.

```heex
<%!-- WRONG: panel fields come up empty, canvas cards don't hover or select --%>
<div :for={tier <- @tiers} class="tier">
  <h3>{tier.name}</h3>
  <p>{tier.price}</p>
</div>

<%!-- CORRECT: container + per-item + per-field markers → full canvas sync --%>
<div data-bs-collection="tiers">
  <div :for={{tier, idx} <- Enum.with_index(@tiers)} data-bs-item={idx} class="tier">
    <h3 data-bs-edit="name">{tier.name}</h3>
    <p data-bs-edit="price">{tier.price}</p>
  </div>
</div>
```

The marker names (`data-bs-collection="tiers"`, field keys `name`/`price`) must match the
prop name and the `item_schema` keys. Item fields are atom-keyed in the HEEx (`tier.name`).

#### Don't pass an editable collection (or prop) as an inline literal

The component's props panel reads its values from the **component instance** (its defaults +
saved overrides), **not** from a literal you hardcode in the `component(...)` call. If a page
embeds `component("related", %{cards: [%{title: "...", href: "/team"}]})` with the cards as an
inline list, the cards render on the public site but the editor panel shows **empty/default
placeholder fields** for them — the user can't see or edit the real values, and the canvas
cards won't hover/select.

For content that **differs per page** (related cards, page-specific feature lists), bind the
prop to a **page `@params` collection** instead of an inline literal: declare the collection
in the page's `params_schema`, then pass it through.

```heex
<%!-- WRONG: cards passed inline → editor panel can't read them, shows empty fields --%>
{component("related", %{cards: [
  %{title: "Our team", blurb: "…", href: "/team"},
  %{title: "Our approach", blurb: "…", href: "/approach"}
]})}

<%!-- CORRECT: page owns the editable collection; component owns the design --%>
{component("related", %{eyebrow: "Related", cards: @params.related_cards})}
```

…with the page's `params_schema` carrying the collection (note `type:"page"` hrefs are page
**IDs**):

```json
{
  "related_cards": {
    "type": "collection", "label": "Related cards",
    "item_schema": {
      "title": {"type": "text",     "label": "Title", "order": 1},
      "blurb": {"type": "textarea", "label": "Blurb", "order": 2},
      "href":  {"type": "page",     "label": "Links to", "order": 3}
    },
    "default": [
      {"title": "Our team", "blurb": "…", "href": "<team_page_id>"},
      {"title": "Our approach", "blurb": "…", "href": "<approach_page_id>"}
    ]
  }
}
```

This is the right pattern whenever the **design is shared but the data is per-page**: a
reusable component for the look, a page `@params` collection for the content. (It mirrors the
contract's `component("pricing-table", %{tiers: @params.tiers})` form.) Reserve inline literals
for truly fixed content that no one edits.

### Option B: inline `data-bs-collection` + `data-bs-item`

If you don't need a reusable component, mark the loop inline. The container gets
`data-bs-collection="name"`, each item gets `data-bs-item={index}` (0-based), and each
editable field inside gets `data-bs-edit`. (The `data-bs-collection` may be on the loop's
parent wrapper with `data-bs-item` on the `:for` children — both that and "both attrs on
each item" group correctly.)

```heex
<%!-- WRONG: bare loop = one opaque block, no per-item editing --%>
<div :for={img <- @params.gallery}>
  <img src={img.url} alt={img.alt} />
</div>

<%!-- CORRECT: each item is its own editable row --%>
<div data-bs-collection="gallery">
  <div :for={{img, idx} <- Enum.with_index(@params.gallery)} data-bs-item={idx}>
    <img data-bs-edit="image" data-bs-edit-type="image" src={img.url} alt={img.alt} />
    <span data-bs-edit="caption">{img.caption}</span>
  </div>
</div>
```

#### `data-bs-item-schema` — the rich inline collection editor (prefer this)

Add `data-bs-item-schema='{…json…}'` to the `data-bs-collection` wrapper to give an inline
collection the **same rich editor a component array gets** — collapsible items,
drag-to-reorder, add/remove, typed sub-field editors. Without it, an inline collection still
works but uses the basic one-item-at-a-time editor. The JSON is the inline analog of a
component prop's `item_schema`: `{ "field": {"type": …, "label": …, "order": …}, … }`.

```heex
<div data-bs-collection="principles"
     data-bs-item-schema='{"title":{"type":"text","label":"Title","order":1},"body":{"type":"textarea","label":"Body","order":2}}'>
  <div :for={{p, idx} <- Enum.with_index(@params.principles)} data-bs-item={idx}>
    <h3 data-bs-edit="title">{p.title}</h3>
    <p data-bs-edit="body">{p.body}</p>
  </div>
</div>
```

Sub-field `type`s: `text`, `textarea`, `page`, `media` (image picker), and `list` /
`collection` for a **repeatable list inside an item** (see next section). Whenever you write
an inline collection whose items have a known shape, add `data-bs-item-schema` — it's the
difference between the basic and the rich editor.

> **Sigil-delimiter trap.** When you pass the schema JSON via an Elixir sigil
> (`data-bs-item-schema={~s({...})}`), the sigil ends at the first **matching close
> delimiter** in the content. A label like `"Bullets (one per line)"` contains `)`, which
> closes `~s(` early → "missing terminator" syntax error on publish. Use a delimiter that
> doesn't appear in the JSON — **`~s[...]`** (brackets) is safest since `{`/`}`/`(`/`)` are
> common in schema text — or avoid parens in labels. The MCP `publish` validates HEEx and
> will reject this, so it fails loudly, not silently.

#### `list` / `collection` — a repeatable list INSIDE a collection item

When an item itself contains a **short repeating list** — a pricing tier's features, a
process phase's bullets, a card's "what's included" — make that sub-field a real repeatable
list. Two shapes:

- **`{"type":"list"}`** — a list of plain strings (the common bullets case). Stored as
  `["…","…"]`. Author maps over it directly. Optional `"item_label"` names the rows
  (`"Bullet"` → "Add Bullet", "1 bullets").
- **`{"type":"collection","item_schema":{…}}`** — a list of objects, one level deep. The
  inner `item_schema` sub-fields are plain `text`/`textarea`/`page` (NOT another nested
  list). Stored as `[{…},{…}]`.

The panel renders an add/remove/reorder editor for the sub-list. **Do NOT** use the old
"textarea, one item per line + `String.split` at render" workaround — `list` replaces it.

```heex
<div data-bs-collection="phases"
     data-bs-item-schema='{"headline":{"type":"text","order":1},"bullets":{"type":"list","label":"Bullets","item_label":"Bullet","order":2}}'>
  <div :for={{phase, idx} <- Enum.with_index(@params.phases)} data-bs-item={idx}>
    <h2 data-bs-edit="headline">{phase.headline}</h2>
    <ul>
      <%!-- bullets is a REAL list. data-bs-edit + data-bs-subindex make each
           row click-to-select on the canvas, like any other field. --%>
      <li :for={{b, bi} <- Enum.with_index(phase.bullets)} data-bs-edit="bullets" data-bs-subindex={bi}>{b}</li>
    </ul>
  </div>
</div>
```

Notes:
- Scalar lists render as `phase.bullets` (atom-keyed item ⇒ list value passes through).
  Object lists keep string keys on the inner maps — author with `f["title"]`, not `f.title`.
- **`data-bs-edit="<list_field>"` + `data-bs-subindex={bi}` on each rendered row are REQUIRED
  for click-to-edit on the canvas.** Iterate with `Enum.with_index(item.field || [])` and put
  both attrs on the element (e.g. the `<li>`). WITHOUT them, clicking the bullet on the canvas
  does **nothing** — the list is only reachable by drilling into the parent item's panel. Since
  the goal is click-the-element-to-edit, treat these as mandatory, not optional. (The schema
  declaring the `list` is necessary but NOT sufficient — the rendered row needs the markers,
  same rule as every other collection/field.)
- The same `list`/`collection` sub-field type works in a **component prop** `item_schema`
  (`props_schema[prop]["item_schema"][subfield]`).

#### `bs_index(idx)` — auto-numbered list items

For an ordered list (steps, principles, rankings), don't store the number ("01", "02") as an
editable field — derive it from position with `bs_index(idx)` so it auto-renumbers on
reorder/add/remove. Options: `bs_index(idx, start: 0)` to start at 0, `bs_index(idx, pad: 2)`
to zero-pad (`01`, `02`). It is **computed, not editable** — do NOT put `data-bs-edit` on the
number element and do NOT include it in `data-bs-item-schema`.

```heex
<div data-bs-collection="principles" data-bs-item-schema='{"title":{"type":"text"},"body":{"type":"textarea"}}'>
  <div :for={{p, idx} <- Enum.with_index(@params.principles)} data-bs-item={idx}>
    <div class="num">{bs_index(idx, pad: 2)}</div>   <%!-- 01, 02, 03… auto --%>
    <h3 data-bs-edit="title">{p.title}</h3>
    <p data-bs-edit="body">{p.body}</p>
  </div>
</div>
```

### Recipe — long legal/policy documents (privacy, terms, HIPAA, …)

A long flowing document (many `<h2>` sections of prose, lists, and links, often with a
sticky table-of-contents sidebar) is the worst case for naive editability: per-paragraph
`data-bs-edit` fields are dozens of fragile fields, and one giant `richtext` blob would strip
the Tailwind layout classes the prose relies on (the richtext+Tailwind trap above). The clean
pattern:

1. **Model the document as a `sections` collection** — each item is
   `{heading:text, anchor:text (optional), body:richtext}`. The `body` richtext holds the
   section's paragraphs/lists/links as inline HTML.
2. **Auto-generate the TOC from the same collection** so it can never desync from the body —
   add a section and it appears in both the sidebar and the document:
   ```heex
   <nav>
     <a :for={s <- @params.sections} href={"#" <> s.anchor}>{s.heading}</a>
   </nav>
   <div data-bs-collection="sections" data-bs-item-schema='{"heading":{"type":"text","order":1},"anchor":{"type":"text","order":2},"body":{"type":"richtext","order":3}}'>
     <div :for={{s, idx} <- Enum.with_index(@params.sections)} data-bs-item={idx} id={s.anchor} style="scroll-margin-top: 6rem;">
       <h2 data-bs-edit="heading">{s.heading}</h2>
       <div class="legal-prose" data-bs-edit="body" data-bs-edit-type="richtext">{raw(s.body)}</div>
     </div>
   </div>
   ```
3. **Put the prose layout styling in page `css_staged`, scoped to a wrapper class** (here
   `.legal-prose`) — NOT in Tailwind classes inside the richtext, which the editor drops on
   save. The richtext only round-trips inline content (`<p>`, `<ul>`/`<ol>`, `<li>`, `<a>`,
   `<strong>`, `<em>`); the wrapper CSS supplies paragraph spacing, list bullets/numbers, and
   link color. Example `css_staged`:
   ```css
   .legal-prose p { margin-top: 1rem; } .legal-prose p:first-child { margin-top: 0; }
   .legal-prose ul { margin-left: 1.5rem; list-style: disc; }
   .legal-prose ol { margin-left: 1.5rem; list-style: decimal; }
   .legal-prose a { color: #1E2A5E; font-weight: 500; }
   ```
4. **Preserve styled callout boxes** (e.g. an "Office for Civil Rights" contact panel) by
   embedding them in the richtext with a CSS class (`.legal-callout`) defined in `css_staged`
   — the box markup lives inside the section `body` and survives editing because its styling
   is class-based, not inline Tailwind.

This gives full editability (every heading + the prose of every section, add/remove/reorder
sections, auto-TOC) with zero fragile coupling. Requires the deployed inline-collection
**richtext sub-field** editor (the `body` sub-field gets the real rich-text widget, not a
raw-HTML input). The `anchor` sub-field is omitted for single-column docs with no TOC.

## Optional — `data-bs-section` for tree grouping

Wrap a page section in `data-bs-section="Label"` to group its `data-bs-edit` fields under
one collapsible node in the editor's element tree. **It is display-only** — it does not
change rendering or how fields are stored. Use it to keep long pages navigable. A schema'd
collection (one with `data-bs-item-schema`) inside the section renders inline in the same
panel alongside the section's scalar fields, so a section + its collection edit as one unit.

Within a section, the editor lists the fields and collections in **DOM order** by default
(both the element tree and the properties panel agree). To override, add an optional
**`data-bs-order="N"`** (integer, lower first) to any `data-bs-edit` or `data-bs-collection`
element — un-annotated nodes fall back to DOM order. Like `data-bs-section`, it is
display-only: no effect on public rendering or stored values.

```heex
<section data-bs-section="Approach steps">
  <div data-bs-collection="approach_steps" data-bs-order="1" data-bs-item-schema="…">…</div>
  <div data-bs-edit="metrics_eyebrow" data-bs-order="2">By the numbers</div>
  <ul data-bs-collection="metrics" data-bs-order="3" data-bs-item-schema="…">…</ul>
</section>
```

```heex
<section data-bs-section="Hero">
  <h1 data-bs-edit="headline" data-bs-edit-type="richtext">…</h1>
  <p data-bs-edit="subhead">…</p>
</section>
```

## Decision rule — which mechanism

- **One-off scalar** text / image / link on a single page → inline element + `data-bs-edit`
  (+ `data-bs-edit-type` when the inferred type is wrong).
- **A section reused across pages, or that should be one tidy tree node** → make it a
  `component()` with a populated `props_schema` (remember the two-call rule).
- **Repeating content** → a component prop with `item_schema` (preferred for reuse), or
  inline `data-bs-collection` + `data-bs-item` **with `data-bs-item-schema`** for the rich
  editor. Never a bare `:for`. Auto-number ordered items with `bs_index(idx)`, don't store it.
  **Always put the collection markers on the rendered DOM, even inside a component.**
- **Shared design + per-page data** (related cards, page-specific lists) → a reusable
  component for the look, bound to a **page `@params` collection** for the content
  (`%{cards: @params.related_cards}`) — never an inline-literal prop.
- **Internal link** → `page_url(...)` + `data-bs-edit-type="page"`, never `href="/slug"`. A
  CTA → `data-bs-edit-type="button"` (or a `type:"button"` component prop).
- **Text with line breaks / inline styling** → `richtext`, and inline `style=` not Tailwind.

## The silent traps (memorize these)

1. **Component with empty `props_schema`** — looks done, edits nothing. `create_component`
   doesn't take `props_schema`; you must call `update_component` after.
2. **Tailwind inside richtext** — `class="…"` is dropped on save; use inline `style="…"`.
3. **`type:"page"` value not a page ID** — every `type:"page"` value (component prop,
   page-level field, AND inline-collection sub-field) is a page **ID**, rendered with
   `page_url(...)`. There is no path-based mechanism anymore. Inline-collection page
   sub-fields render `href={page_url(item.field)}`, same as component arrays. (Legacy path
   values still resolve via the back-compat shim, but author new values as IDs.) Wrong format
   → picker shows "— Select a page —". See the internal-links section.
4. **Props with no `order` key** — list in random order; set `order` on every prop.
5. **Collection prop rendered with a bare `:for` (no `data-bs-collection`/`data-bs-item`/
   `data-bs-edit` on the rendered DOM)** — the panel shows empty placeholder fields and the
   canvas cards don't hover or select. A populated `item_schema` is necessary but **not
   sufficient**; the rendered loop needs the markers too.
6. **Editable prop passed as an inline literal** in the `component(...)` call — the panel
   reads the instance, not the literal, so the fields show as empty defaults. Bind per-page
   data to a page `@params` collection (`%{cards: @params.related_cards}`) instead.
7. **Elixir empty-string truthiness in `:if` guards / `||` fallbacks.** In Elixir `""` is
   **truthy**, so `"" || @fallback` returns `""` (not `@fallback`), and `<a :if={@label}>`
   renders even when `@label` is `""`. Guard explicitly on non-empty:
   `:if={@label && @label != ""}` and `value = if(x in [nil, ""], do: default, else: x)`.
   Getting this wrong silently hides or shows whole blocks (e.g. a CTA that vanishes, or an
   empty button that renders).

## Pre-ship editability checklist

- [ ] Every page has at least one editable node (`data-bs-edit` and/or a component).
- [ ] Every reused section is a `component()` with a **non-empty** `props_schema`.
- [ ] Every repeating block is a collection/array (component `item_schema`, or inline
      `data-bs-collection`/`data-bs-item`) — no bare `:for` loops for editable content.
- [ ] Every collection's **rendered DOM** carries the markers — `data-bs-collection` on the
      container, `data-bs-item={idx}` per item, `data-bs-edit` per field — including inside
      components (a populated `item_schema` alone does NOT give canvas hover/select).
- [ ] No editable prop is passed as an inline literal in a `component(...)` call; per-page
      data is bound from a page `@params` collection.
- [ ] Inline collections with a known item shape declare `data-bs-item-schema` (rich editor).
- [ ] Link/CTA props in reusable components run through the `resolve` href helper (accept a
      page ID *or* a literal path/anchor/tel).
- [ ] `:if` guards and `||` fallbacks test non-empty (`x && x != ""`), not just truthiness —
      `""` is truthy in Elixir.
- [ ] Ordered-list numbers use `bs_index(idx)`, not a stored editable field.
- [ ] Internal links use `page_url(...)` + `data-bs-edit-type="page"`; CTAs use `button`.
- [ ] Richtext fields use inline `style=`, not Tailwind classes.
- [ ] `data-bs-edit-type="page"` is on every internal-link `<a>`.
- [ ] `<a>` tags with icons/child markup have a `data-bs-edit-target` on the text child.
- [ ] Every prop has an `order` key; `type:"page"` defaults are page IDs.

## Verified source references

Authored against the BrightSite server as of this writing. Key code paths:
`visual_editor_renderer.ex` (iframe script: `inferEditType`, `getScope`,
`getCollectionContext`); `visual_editor_components.ex` (panel: array/collection branch,
`order` sort, empty-`props_schema` gate); `mcp/server.ex` (`create_component` /
`update_component` / `create_page` tool params). If a field is renamed in the MCP schema,
treat that as the higher authority and flag it.
