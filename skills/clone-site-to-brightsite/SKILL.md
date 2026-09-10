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

**You have not finished until you have looked at your own output next to the
original.** A page that renders without errors is not a page that matches. Phase
5 is not optional, and the report must state what still differs.

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

## Phase 4: Assembly

Create or update each page (`create_page` / `update_page`), assign the layout,
then `publish_page`. Content only reaches the live site on publish.

Delete leftover starter pages that are not part of the source site.

Create redirects for any source path whose shape changed.

## Phase 5: Visual QA — mandatory

Screenshot your page and the source at the same viewport, then **look at both**.

```bash
agent-browser set viewport 1440 900
agent-browser open "<source-url>";  agent-browser screenshot /abs/shots/src.png
agent-browser open "<brightsite-url>"; agent-browser screenshot /abs/shots/mine.png
```

Read both images. Compare top to bottom: hero treatment, nav, type scale and
tracking, section order, spacing, imagery. Repeat at 390.

Then run the checks a screenshot can miss:

```javascript
// broken images — naturalWidth 0 means it failed to load
Array.from(document.images).map(i => ({src: i.currentSrc, ok: i.naturalWidth > 0}))
// empty content regions
document.body.innerText.includes('No products yet')
```

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
- **Do not verify only the top of the page.** Scroll past the first section and
  compare again; a fixed backdrop and a normal one look identical until you do.
- **Do not report success on a page you have not looked at.**
