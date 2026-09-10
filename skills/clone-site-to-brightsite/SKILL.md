---
name: clone-site-to-brightsite
description: Rebuild an existing website as a BrightSite site with pixel-accurate fidelity — extracts real computed CSS, assets, content and behaviour from the live source, then authors layout, components and pages over MCP and diffs the result against the original until it matches. Use when the user says "clone this site", "rebuild this site on BrightSite", "make it look exactly like the old site", "match the existing design", or when migrating a site whose look must be preserved. The WordPress and WooCommerce migration skills call this for the visual half.
---

# Clone a site into BrightSite

Rebuild a live website as a BrightSite site that looks the same to a visitor and
stays editable by the client afterwards.

This is not "look at the site and write some HEEx". It is a measured pipeline:
every colour, size and spacing value comes from `getComputedStyle()` on the real
page, every asset is downloaded and rehosted, and the finished page is
screenshotted and compared against the original before you call it done.

Methodology adapted from the MIT-licensed
[ai-website-cloner-template](https://github.com/JCodesMore/ai-website-cloner-template).

## The rule that matters most

**Every check asks "does this match the source?", never "does this render?"**

That distinction is the whole job. A page that renders without errors, with no
broken images, no console noise and every section present, can still be wrong in
every way that matters: a menu missing an item, an icon the source never had, a
badge the wrong shape, a heading at 13px where the source is 11px. "It renders"
passes on all of those. "It matches" does not.

So a check is only worth running if it can fail while the page looks fine. In
practice that means: measure the source, measure the clone, compare the two
numbers. A check with only one side is not verification.

Concretely — never write the check on the left; write the one on the right:

| Not this | This |
|---|---|
| the nav has links | the nav has *the same links, in the same order*, as the source |
| the badge renders | the badge is 14x14, `rgb(229,229,229)`, radius 2px — as measured |
| the hero has a photo | the hero is `100vh` with a pinned backdrop, like the source |
| the footer is visible | the footer has *the same three columns* with the same items |

You have not finished until you have put your output next to the original and
compared them. Phase 5 is not optional, and the report must state what still
differs.

## Pre-flight

1. **Browser automation is required.** This skill cannot work without it. Use
   `agent-browser` (or the project's configured browser tool). Verify it opens
   the target URL before doing anything else.
2. **Account.** Get the BrightSite `account_id` (`list_accounts`).
3. **A brand-new account must be onboarded first.** Call `complete_onboarding`
   before writing any content — otherwise the first visit to the dashboard runs
   the setup wizard and seeds the starter template over everything you built.
4. **Target URLs.** One or more pages. Confirm each responds.
5. **Working directory** for artifacts: screenshots, spec files, extraction
   output. Keep them; they are the audit trail when something looks wrong.

## Scope

- **In scope:** layout, spacing, typography, colour, imagery, real text, section
  structure, responsive behaviour, hover/scroll behaviour.
- **Out of scope unless asked:** third-party widgets with no BrightSite
  equivalent (booking, donations, wishlists, comparison tables), backend
  behaviour, analytics.
- **Never invent copy.** Every string comes from the source page. If a section
  cannot be reproduced, say so in the report — do not paper over it with
  plausible-looking filler.

## Guiding principles

**Completeness beats speed.** If any value in a section is guessed — a colour, a
font size, a padding — extraction was not finished. Spend the extra minute.

**Small pieces, exact results.** Given a whole page at once, you approximate.
Given one section with measured values, you match it. If a section has several
distinct sub-parts with their own styling and behaviour, treat them separately.

**Real content, real assets.** Extract the actual text and download the actual
images. Generate nothing. A clone with placeholder copy is a mockup.

**Foundation first.** Fonts, palette, layout CSS and assets must exist before
any section is authored. Everything else depends on them.

**Extract how it looks AND how it behaves.** A page is not a screenshot.
Capture the computed CSS *and* what changes, what triggers the change, and how
it transitions.

**Identify the interaction model before building.** Building a click-driven UI
for a scroll-driven section is a rewrite, not a CSS tweak. Scroll and wait
first; click only after.

**Extract every state, not just the default.** A header at scroll 0 and the
same header scrolled are two specs. So is a hovered card, or a tab that is not
currently active.

## Phase 1: Reconnaissance

Do this before writing a single line of HEEx.

### Screenshots

Capture the source at three widths and keep them as the reference:

```bash
agent-browser set viewport 1440 900
agent-browser open "<source-url>"
agent-browser screenshot /abs/path/shots/src_<page>_1440.png
agent-browser set viewport 768 1024   # tablet
agent-browser set viewport 390 844    # mobile
```

Paths must be absolute — a relative path fails.

### Global tokens

Extract, do not guess:

```javascript
JSON.stringify({
  fonts: [...new Set([...document.querySelectorAll('h1,h2,h3,p,a,body,button')]
           .map(e => getComputedStyle(e).fontFamily))],
  webfonts: [...document.querySelectorAll('link[href*="fonts."]')].map(l => l.href),
  bodyColor: getComputedStyle(document.body).color,
  bodyBg: getComputedStyle(document.body).backgroundColor,
  headingColor: getComputedStyle(document.querySelector('h1,h2')||document.body).color,
  logo: (document.querySelector('img[class*=logo],.custom-logo')||{}).src,
  favicons: [...document.querySelectorAll('link[rel*=icon]')].map(l => l.href)
})
```

The brand colours of a themed CMS site usually live in an **inline `<style>`
block** written by the theme customiser, not in the stylesheet. Read those
blocks — that is where the real palette is.

### Interaction sweep

Static screenshots hide half of a page. Before specifying anything:

- **Scroll** slowly top to bottom, and watch for each of these — the list is
  not exhaustive, and anything the page does that is not on it still counts:
  - a header that shrinks, changes background or gains a shadow past a threshold
  - elements animating into view on entry (fade-up, slide-in, stagger delays)
  - `scroll-snap-type` on a container
  - parallax layers, or a **pinned backdrop the content scrolls over**
  - scroll-driven progress bars or opacity transitions
  - a sidebar or tab indicator that switches by itself as content passes
    (IntersectionObserver, *not* click handlers)
  - a smooth-scroll library — check for `.lenis` or `.locomotive-scroll`;
    default browser scrolling feels obviously different
  - `animation-timeline` in the CSS
  - an auto-advancing carousel: **wait several seconds without touching
    anything**, because a carousel and a static photo are identical in a
    screenshot
- **Hover** over nav items, buttons, cards. Record what changes.
- **Click** anything that looks interactive — tabs, pills, arrows. Record the
  content of *each* state, not just the default.
- **Resize** to 768 and 390. Record what stacks, what disappears.

Write the findings down. A hero that is a rotating photo carousel and a hero
that is one static photo need different HEEx, and you cannot tell them apart
from one screenshot.

### Page topology

List every section top to bottom with a working name, its height, and whether
it is flow content or a fixed overlay.

**Measure the height against the viewport, not in isolation.** A section that
reports 900px at a 900px viewport is almost certainly `100vh`, and hardcoding
900px breaks on every other screen. Load the page at two viewport heights and
compare:

```javascript
JSON.stringify({vh: window.innerHeight,
  hero: Math.round(document.querySelector('SECTION').getBoundingClientRect().height)})
```

Same number at both heights → a fixed pixel height. Tracks the viewport →
`100vh`.

**Check `position` on the section's children, not only the section.** A hero
whose section is `position: relative` can contain a `position: fixed` slider —
that is a pinned backdrop the page scrolls over, and it is invisible in both a
screenshot and a section-level style dump:

```javascript
JSON.stringify([...document.querySelectorAll('SECTION *')]
  .map(e => ({cls:(e.className||'').toString().slice(0,40), pos:getComputedStyle(e).position}))
  .filter(x => x.pos === 'fixed' || x.pos === 'sticky'))
```

Then list the topology:

```javascript
JSON.stringify([...document.querySelectorAll('body section, body > div > section')]
  .map(e => ({ cls: (e.className||'').toString().slice(0,60),
               h: Math.round(e.getBoundingClientRect().height),
               pos: getComputedStyle(e).position }))
  .filter(x => x.h > 40))
```

## Phase 2: Foundation

Sequential, and everything else depends on it.

1. **Assets.** Enumerate every image on the page — including background images
   and absolutely-positioned overlays, which are easy to miss:

   ```javascript
   JSON.stringify({
     images: [...document.querySelectorAll('img')].map(i => ({
       src: i.currentSrc || i.src, alt: i.alt, w: i.naturalWidth, h: i.naturalHeight })),
     backgrounds: [...document.querySelectorAll('*')]
       .map(e => getComputedStyle(e).backgroundImage)
       .filter(b => b && b !== 'none')
   })
   ```

   Take image URLs **from the rendered page**, never construct them from a
   filename — the same filename exists under several dated folders on a
   WordPress site and you will silently fetch the wrong one, or a 404 page.

   Upload each through the two-step MCP flow: `request_upload` → HTTP `PUT` the
   raw bytes → `complete_upload`. **Check the bytes are an image first**
   (`FF D8` JPEG, `89 50 4E 47` PNG, `RIFF…WEBP`); a 404 from the source CMS is
   an HTML page, and uploading it produces a media record that looks fine in a
   list and is broken on the page.

   Keep a map of source URL → BrightSite media id. You need it in every
   subsequent phase.

2. **Layout + theme CSS.** Author the site chrome into the layout
   (`update_layout` then `publish_layout`):
   - webfont `<link>` tags in the head
   - CSS custom properties for the extracted palette
   - the section/typography classes the pages will use

   Put shared CSS in the **layout**, not on individual pages, so every page
   inherits it.

3. **Site identity.** `update_site_identity` with the real title, tagline and
   logo. **Clear the starter placeholders** — an untouched account ships with a
   fake address, phone and email that otherwise end up in your JSON-LD.

4. **Chrome.** Nav and footer are builtin components: `update_builtin_content`
   with `kind: "navigation"` / `"footer"` and a `values` object (the param is
   `values`, not `props`). Nav items take `page` (a page id) or `url`.

## Phase 3: Section specs

For each section in the topology, extract → write a spec → build. Do not skip
the spec: it is what stops a section being built from memory.

### Extract

Run the full computed-style walk on the section's container. Store the JSON:

```javascript
(function(sel){
  const el=document.querySelector(sel); if(!el) return JSON.stringify({error:'not found'});
  const props=['fontSize','fontWeight','fontFamily','lineHeight','letterSpacing','color',
   'textTransform','textAlign','backgroundColor','backgroundImage','backgroundSize',
   'backgroundPosition','padding','paddingTop','paddingBottom','paddingLeft','paddingRight',
   'margin','marginTop','marginBottom','width','height','maxWidth','minHeight','display',
   'flexDirection','justifyContent','alignItems','gap','gridTemplateColumns','borderRadius',
   'border','boxShadow','position','top','left','zIndex','opacity','transform','transition',
   'objectFit','filter','backdropFilter'];
  const grab=e=>{const cs=getComputedStyle(e),o={};
    props.forEach(p=>{const v=cs[p];
      if(v&&v!=='none'&&v!=='normal'&&v!=='auto'&&v!=='0px'&&v!=='rgba(0, 0, 0, 0)')o[p]=v;});
    return o;};
  const walk=(e,d)=>d>4?null:({tag:e.tagName.toLowerCase(),
    classes:(e.className||'').toString().split(' ').slice(0,5).join(' '),
    text:e.childNodes.length===1&&e.childNodes[0].nodeType===3?e.textContent.trim().slice(0,300):null,
    styles:grab(e),
    img:e.tagName==='IMG'?{src:e.currentSrc||e.src,alt:e.alt,w:e.naturalWidth,h:e.naturalHeight}:null,
    children:[...e.children].slice(0,20).map(c=>walk(c,d+1)).filter(Boolean)});
  return JSON.stringify(walk(el,0));
})('SELECTOR')
```

For anything with more than one state (scrolled header, hover card, active tab),
capture **both** states and record the difference plus the transition.

Copy the text content verbatim. "SHINE BRIGHT" is not "We Shine Bright".

### Write the spec

One file per section, before building:

```markdown
# <Section> spec
- Source URL / selector:
- Screenshot:
- Interaction model: static | hover | scroll-driven | time-driven (carousel)
- Becomes: inline page section | BrightSite component (slug)

## Structure
<what contains what>

## Computed styles
### Container
padding: … / background: … / minHeight: …
### Heading
fontFamily: … / fontSize: … / letterSpacing: … / textTransform: … / color: …
(exact values, from getComputedStyle)

## States & behaviours
For each: trigger, state A, state B, transition.
- **Trigger:** scroll past 50px / hover / click on X / auto every N seconds
- **State A (before):** background: transparent, height: 140px, padding: 15px 0
- **State B (after):** background: rgba(44,62,80,.9), height: 130px, padding: 10px 0
- **Transition:** background, padding 0.4s ease-in-out
Write "N/A" only after checking — even a footer usually has link hover states.

## Content (verbatim)
…

## Assets
source URL → media id

## Responsive
1440: … / 768: … / 390: … / breakpoint ≈ …

## Editability
data-bs-edit fields: …
params_schema collections: …
```

### Inline section or component?

- **A BrightSite component** when the section repeats across pages or recurs
  many times (site chrome, a card used on several pages, a product strip).
  Create with `create_component` and a real `props_schema`, embed with
  `component("slug")`.
- **Inline in the page** when it is a one-off. Do not manufacture a component
  for a section that appears once.

Either way, wrap the section in `data-bs-section="Label"` so it groups into one
collapsible node in the visual editor's element tree. A long page without
grouping is unusable for the client.

### A blanket background rule will repaint your dark sections

Giving everything after a pinned hero an opaque background is how you stop the
backdrop showing through — but a rule like
`main > *:not(.hero) { background: #fff }` also repaints a dark footer or any
other dark section that lives inside the page. The text stays the colour you set
and becomes invisible. Re-assert the background on every section that is not the
page's default colour.

### Use the builtin navigation and footer

BrightSite ships **builtin components** for site chrome, and the layout should
embed them:

```
{component("navigation")}
{@inner_content}
{component("site-footer")}
```

They are what the client edits in the dashboard, so a clone that hand-rolls its
own nav takes that away. Populate them over MCP with
`update_builtin_content` (the param is `values`, not `props`):

```json
{"kind": "navigation",
 "values": {"show_logo": true, "sticky": true,
   "items": [{"label": "Home", "page": "<page-id>", "url": "", "children": []},
             {"label": "Shop", "page": "", "url": "/collections/x", "children": []}]}}
```

`page` for an internal page (survives slug changes), `url` for anything else.
Extract the **full** menu from the source — every item, in order. A menu that is
missing entries is the same failure as a section built from memory.

**Know what the builtins cannot express, and what the helpers can.** The nav
builtin takes items, buttons, a logo and a sticky flag; it has no slot for a
search icon, an account link or a cart badge. That does **not** mean the site
cannot have them — the storefront template helpers do:

| helper | gives you |
|---|---|
| `cart_url()` | the cart page URL |
| `cart_count()` | items in this viewer's cart, `0` when empty |
| `account_url()` | `/account` when signed in, `/account/sign-in` otherwise |
| `account_label()` | "Sign in" or the customer's name |
| `commerce?()` | whether this site has a store — wrap the block in it |

Author that cluster in the **layout**, next to `component("navigation")`, the
way the starter storefront layout does. Check an existing storefront site's
layout before concluding a thing is impossible.

The footer builtin takes a copyright line and simple columns; it cannot do
WooCommerce-style widget columns with product thumbnails and prices, so that one
does need rebuilding as a section. Check the builtin first, and never replace one
just because styling it looked easier.

### Chrome lives in the layout, not the page

Nav and footer are rendered by the layout, outside the page's HEEx. A wrapper
added inside a page therefore does **not** contain them, so any stacking
context, background or z-index a pinned backdrop needs must be applied to
`header` and `footer` from the **layout CSS**. Forgetting this is how a footer
disappears behind a fixed hero while still being present in the DOM.

The builtin nav and footer also ship their own Tailwind utilities (`bg-white`,
`border-zinc-200`, `mt-24`, an `h-8` logo). Props do not reach those. Override
them from the layout CSS against the rendered markup, and note the header is
nested inside LiveView wrapper divs — a `body > header` selector will not match,
so use a descendant selector.

### Build it editable

Fidelity is only half the job — the client has to be able to change the thing
afterwards:

- `data-bs-edit="name"` on every heading, paragraph, image and link.
- `data-bs-edit-type="richtext"` where the text carries `<br>` or inline
  emphasis; inside richtext use inline `style=`, not Tailwind classes, which are
  stripped on save.
- `params_schema` collections for anything repeating (team members, feature
  cards, testimonials) with `:for` in the template. Never hardcode a repeating
  list.
- Internal links: `href={page_url("id")}` plus `data-bs-edit-type="page"`.

### Before you build a section, check every box

If you cannot tick one, go back and extract more.

- [ ] A spec file exists for this section with every field filled
- [ ] Every CSS value came from `getComputedStyle()`, none estimated
- [ ] The interaction model is identified (static / hover / scroll / time)
- [ ] Heights checked at two viewport heights — is it px or `100vh`?
- [ ] The section's descendants were swept for `position: fixed` / `sticky`
- [ ] For stateful elements: both states captured, with the transition
- [ ] Every image found, including background images and overlays
- [ ] Responsive behaviour recorded at 1440 and 390
- [ ] Text is verbatim, not paraphrased
- [ ] `data-bs-edit` fields and any collections are planned

### After you build a section, before you move on

Extraction checks are not enough — everything above can pass and the built
section still not match. Diff it against the source now, while the spec is in
front of you, rather than discovering it in Phase 5 or having the user find it:

- [ ] The element list matches: same items, same order, nothing added
- [ ] Every value in the spec appears in the built section (spot-check the
      three most visible: font size, tracking, colour)
- [ ] Nothing was invented — no icon, link or block the source does not have
- [ ] Nothing moved between regions relative to the source
- [ ] Screenshot the section on both sides and compare the two images

That fourth box is the one that keeps being missed. Reorganising the source's
content because the new arrangement seems tidier is not cloning it.

## Phase 4: Assembly

Create or update each page (`create_page` / `update_page`), assign the layout,
then `publish_page`. Content only reaches the live site on publish.

Delete leftover starter pages that are not part of the source site.

Create redirects for any source path whose shape changed.

## Phase 5: Visual QA — mandatory

**Every check in this phase compares two sides.** Run each one against the
source *and* the clone and diff the results. A single-sided check — "the nav
rendered", "no broken images" — tells you the page is not crashing, which is not
the question.

Screenshot your page and the source at the same viewport, then **look at both**.

```bash
agent-browser set viewport 1440 900
agent-browser open "<source-url>";  agent-browser screenshot /abs/shots/src.png
agent-browser open "<brightsite-url>"; agent-browser screenshot /abs/shots/mine.png
```

Read both images. Compare top to bottom: hero treatment, nav, type scale and
tracking, section order, spacing, imagery. Repeat at 390.

Then run the checks a screenshot can miss. **Run each on the source first, then
on the clone, and compare** — the source's numbers are the target, not zero and
not "some":

```javascript
// Run on BOTH. Every count should match the source's count.
JSON.stringify({
  brokenImages: [...document.images].filter(i => !i.naturalWidth).length, // clone: 0
  images:       document.images.length,        // must match the source
  navLinks:     [...document.querySelectorAll("header nav a")].map(a => a.textContent.trim()),
  sections:     document.querySelectorAll("main > *").length,
  h2s:          [...document.querySelectorAll("h2")].map(h => h.textContent.trim())
})
```

A clone with fewer images, fewer sections or a shorter heading list than the
source is missing content, however cleanly it renders.

**A screenshot of the top of the page proves almost nothing.** Two failures
survive it every time, so test them explicitly:

*Viewport-relative height* — screenshot at 1440x900 and again at 1440x700, and
compare the hero height to `window.innerHeight` in both. A hero hardcoded to
900px looks perfect in a 900px-tall screenshot and is wrong on every other
screen:

```javascript
JSON.stringify({vh: window.innerHeight,
  hero: Math.round(document.querySelector('.hero').getBoundingClientRect().height)})
```

*Scroll behaviour* — scroll past the first section and re-measure. If the source
pins a backdrop, its rect stays at top 0 while `scrollY` grows. Compare the same
two numbers on the source and on the clone:

```javascript
// after: agent-browser scroll down 700
(function(){ const r = document.querySelector('.hero-backdrop').getBoundingClientRect();
  return JSON.stringify({scrollY: Math.round(window.scrollY),
                         top: Math.round(r.top), pinned: Math.abs(r.top) < 2}); })()
```

*The region matches element for element.* Screenshot the same region on both
sides and compare the two images directly — this is the check that catches a
missing menu item, an icon that exists only in your version, or a badge with the
wrong shape. Verifying that "the nav renders" is not verifying that the nav
matches.

For any region with a fixed set of parts (a menu, an icon row, a footer column
set), enumerate both sides and diff the lists:

```javascript
// run on the SOURCE, then on the CLONE, and compare the output
JSON.stringify({
  menu: [...document.querySelectorAll("NAV_SELECTOR a")].map(a => a.textContent.trim()),
  icons: document.querySelectorAll("NAV_SELECTOR svg, NAV_SELECTOR i").length
})
```

Counts that differ are the bug. Do not reconcile them by reasoning about what
seems right — the source list is the answer.

*Every region is present AND visible* — an element can render correctly and
still be invisible because something paints over it. `querySelector` finding it
is not proof. Hit-test the middle of each major region and confirm the topmost
element there is the one you expect:

```javascript
(function () {
  return JSON.stringify(["header", "main", "footer"].map(sel => {
    const el = document.querySelector(sel);
    if (!el) return { sel, present: false };
    const r = el.getBoundingClientRect();
    const hit = document.elementFromPoint(window.innerWidth / 2, r.top + r.height / 2);
    return { sel, present: true, h: Math.round(r.height),
             visible: !!(hit && hit.closest(sel)) };
  }));
})()
```

Any region with `visible: false` is being covered — usually by a pinned
backdrop whose stacking context the region never joined.

*No seams over a pinned backdrop* — measure the gap between adjacent regions.
A transparent margin between them shows the backdrop through:

```javascript
(function () {
  const m = document.querySelector("main").getBoundingClientRect();
  const f = document.querySelector("footer").getBoundingClientRect();
  return JSON.stringify({ gap: Math.round(f.top - m.bottom) });
})()
```

**Use a full-page screenshot** (`agent-browser screenshot <path> --full`).
A viewport screenshot captures whatever scroll position the screenshot process
happens to be at, which is *not* necessarily where a separate `eval` call
scrolled to — measurements and pixels can disagree, and the pixels are right.
A full-page capture removes the question. The footer is the region most often
broken and least often looked at.

Two caveats on the full-page capture: `position: fixed` layers (a pinned hero,
a fixed nav) do not composite into it, so the hero reads as blank and the nav
appears mid-page. Verify those from the viewport screenshot instead.

**When a measurement and a screenshot disagree, believe the screenshot.**
`getComputedStyle` reporting a visible colour proves the rule matched, not that
the element is legible: white text on a background that another rule repainted
white measures perfectly and reads as nothing. Sample the actual pixels where
the element should be if you need certainty.

Take the comparison screenshots **scrolled**, not only at the top.

For each difference: fix the HEEx or CSS, republish, re-screenshot. Cap at about
three rounds, then report whatever still differs rather than looping forever.

**Common causes when an image is broken:**
- extension mismatch — the file was stored `.jpeg` but the URL asks for `.jpg`
- the local image proxy is serving a stale mount (restart it) — a dev-only issue
- the upload silently stored an HTML error page instead of an image

## Phase 6: Report

- Source URL → BrightSite page for each page
- Sections built, and which became components vs inline
- Assets rehosted, and the count of remaining references to the source domain
  (**should be zero** — grep the published HEEx and any imported HTML)
- Screenshot pairs, and the differences that remain
- What was deliberately not cloned, and why

## Anti-patterns

- **Do not write HEEx from memory or from a screenshot glance.** Every value
  comes from `getComputedStyle()`.
- **Do not skip Phase 5.** A page that renders is not a page that matches.
- **Do not paraphrase copy.** Take the exact strings.
- **Do not construct asset URLs from filenames.** Read them off the page.
- **Do not upload bytes without checking they are an image.**
- **Do not build a click-driven UI for a scroll-driven or auto-playing section.**
  Establish the interaction model by scrolling and waiting *before* clicking.
- **Do not miss layered images.** One visual can be a background plus a
  foreground overlay; enumerate every `img` and background-image in the
  container.
- **Do not leave the starter placeholders** (Acme Co., 123 Main Street) in site
  identity.
- **Do not build a section as a component just because it is big.** Components
  are for things that repeat.
- **Do not hardcode a height that is really `100vh`.** Measure at two viewport
  heights before writing a pixel value.
- **Do not extract only the section element.** Its children carry `position:
  fixed`/`sticky`, which is where pinned backdrops and parallax live.
- **Do not trust a viewport screenshot's scroll position.** Capture full-page,
  and check fixed layers separately.
- **Do not hand-roll a nav or footer before checking the builtins.** They are
  what the client edits; replacing them silently removes that.
- **Do not add an element the source does not have.** A clone with an extra
  account icon is as wrong as one missing a menu item, and it is easier to miss
  because nothing looks broken.
- **Do not move an item between regions.** If the source puts "My Account" in
  the menu, it goes in the menu — not into an icon row because that felt tidier.
- **Do not conclude a builtin "cannot do X" without checking the helpers and an
  existing storefront layout.** A cart badge and account link are helper calls,
  not missing features.
- **Do not ship a partial menu.** Extract every item from the source, in order.
- **Do not let a blanket background rule repaint a dark section.**
- **Do not treat "the element exists" as "the element is visible."** Hit-test
  it; a pinned backdrop hides regions that query perfectly.
- **Do not verify only the top of the page.** Scroll past the first section and
  compare again; a fixed backdrop and a normal one look identical until you do.
- **Do not report success on a page you have not looked at.**
