---
name: clone-site-to-brightsite
description: Rebuild an existing website as a BrightSite site with pixel-accurate fidelity — extracts real computed CSS, assets, content and behaviour from the live source, then authors layout, components and pages over MCP and diffs the result against the original until it matches. Use when the user says "clone this site", "rebuild this site on BrightSite", "make it look exactly like the old site", "match the existing design", or when migrating a site whose look must be preserved. Detects a blog or shop on the source and rebuilds the blog as a real BrightSite blog (listing, post card, post template, settings) rather than a page per post; hands the catalog to commerce-catalog.md. Use it for WooCommerce and WordPress site moves too — for a posts-only import from a WXR export file, use migrate-wordpress-blog.
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
4. **Target URLs.** One or more pages. Normalise each and confirm each
   responds. If any is unreachable, stop and ask rather than cloning what you
   can reach.
5. **Inventory what already exists** in the account before writing anything:
   `list_pages`, `list_components`, `list_layouts`, `list_media_folders`.
   Distinguish the untouched starter template from pages a human or an earlier
   run authored. A new clone target is not permission to overwrite them.
6. **Working directory** for artifacts: screenshots, spec files, extraction
   output. Keep them; they are the audit trail when something looks wrong.
7. **Write the output plan** (below) and get it agreed before Phase 1.

## The output plan

Do this even for a single page; it takes a minute and it is what stops the
second page of a job overwriting the first.

List, for every target URL:

| source URL | slug | layout | components it will own | media folder |
|---|---|---|---|---|

Rules:

- **Preserve the source's path.** `/about-us/` becomes the slug `about-us`, not
  `about`. Inbound links, the client's bookmarks and the old sitemap all assume
  the original path. Where a path genuinely must change, note it now so Phase 4
  creates the redirect.
- **URLs differing only by query string or fragment are one page.** Decide
  explicitly whether the difference is a distinct page, a tab state, or nothing.
- **Namespace per page.** Give each page its own artifact directory
  (`shots/<slug>/`, `specs/<slug>/`) and its own media folder. Two pages writing
  `hero.png` into one folder is a silent overwrite.
- **Name components for the site, not the page** — a component is shared by
  definition, so it belongs to the site's namespace, not to whichever page
  happened to need it first.
- **If a planned slug already exists**, stop and ask: update that page, pick
  another slug, or skip it. Never delete a page you did not create in this run.
- **One source site per BrightSite site.** Two different origins need two
  accounts. They will disagree about fonts, palette, layout and chrome, and
  merging them means one of them ends up wrong.

## Order of work for a multi-page clone

1. Phase 1 recon on **every** target page first, so the topology and the shared
   chrome are known before anything is authored.
2. Phase 2 foundation **once**, sequentially — fonts, palette, layout, nav,
   footer, site identity. Every page depends on it, so it cannot be built twice
   in parallel.
3. Only then Phase 3 per page. Pages may be built in parallel once the
   foundation is fixed and each page's artifacts and media folder are separate.
4. Phase 5 QA per page, plus one pass over the shared chrome.

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

**Small pieces, exact results.** Given a whole page at once, you approximate:
spacing gets eyeballed, font sizes get guessed, and the result is "close enough"
and visibly wrong. Given one section with measured values, you match it.

Judge each section's complexity and split accordingly. A banner with a heading
and a button is one unit. A section with three card variants, each with its own
internal layout and hover state, is four units: one per variant plus the
wrapper. When in doubt, smaller.

**The complexity budget.** If a section's spec runs past roughly 150 lines, the
section is too big to build in one go — split it. This is a mechanical check.
Do not override it with "but it is all one section"; that is exactly the
reasoning that produces an approximated build.

**Real content, real assets.** Extract the actual text and download the actual
images. Generate nothing. A clone with placeholder copy is a mockup.

**Foundation first.** Fonts, palette, layout CSS and assets must exist before
any section is authored. Everything else depends on them.

**Extract how it looks AND how it behaves.** A page is not a screenshot.
Capture the computed CSS *and* what changes, what triggers the change, and how
it transitions.

**Identify the interaction model before building.** Building a click-driven UI
for a scroll-driven section is a rewrite, not a CSS tweak — it is the single
most expensive mistake in cloning. Answer definitively, before specifying any
interactive section: is it driven by scroll, click, hover, time, or a
combination?

Determine it in this order, and do not shortcut it:

1. **Do not click first.** Scroll through the section slowly and watch whether
   anything changes on its own.
2. **Wait several seconds without touching anything.** A carousel that advances
   on a timer is identical to a static photo in a screenshot.
3. If something changed by itself, it is scroll- or time-driven. Find the
   mechanism: `IntersectionObserver`, `scroll-snap`, `position: sticky`,
   `animation-timeline`, or a scroll listener.
4. Only if nothing changes on its own, click and hover to test for
   click/hover-driven behaviour.
5. Write the answer into the spec as a line of its own: "INTERACTION MODEL:
   scroll-driven, IntersectionObserver at ~30% viewport".

A sticky sidebar whose items highlight as content scrolls past is a different
thing from a tab bar you click. They look the same in a screenshot.

**Extract every state, not just the default.** A header at scroll 0 and the
same header scrolled are two specs. So is a hovered card, or a tab that is not
currently active. Whatever is on screen at page load is one state out of
several, and it is the only one you get for free.

For tabbed or otherwise switchable content:

- Click **each** tab, pill or arrow in turn.
- Extract the text, images and card data that appear for **each** state, and
  record which content belongs to which state.
- Record the transition between states (opacity, slide, fade, duration, easing).

For scroll-dependent elements:

- Capture the computed styles at scroll 0.
- Scroll past the trigger and capture the same element again.
- **Diff the two.** The list of properties that changed *is* the behaviour spec.
- Record the trigger precisely — a scroll offset in px, or an intersection
  ratio — and the transition CSS.

**Check what a visual actually is before rebuilding it.** A section that looks
like a styled HTML block may be a `<video>`, a Lottie animation or a `<canvas>`.
Building an elaborate HTML mockup of a video's contents is wasted work that can
never match. Check the container's tag names first:

```javascript
JSON.stringify([...document.querySelectorAll('SECTION video, SECTION canvas, SECTION lottie-player, SECTION [data-lottie]')]
  .map(e => ({tag: e.tagName, src: e.currentSrc || e.src || '',
              poster: e.poster || '', autoplay: e.autoplay, loop: e.loop})))
```

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

### What kind of site is this? Fingerprint before you plan

A source site is rarely just pages. If it has a blog or a shop, cloning the
pages alone hands back a site that looks right and is missing most of its
content — and nothing about the result looks broken, so it ships.

Run this **before** the output plan is fixed, because what you find changes the
plan. From the shell:

```bash
S="https://SITE"
curl -sI  "$S/wp-json/wp/v2/posts?per_page=1"   | grep -i '^x-wp-total'   # post count
curl -sI  "$S/wp-json/wp/v2/product?per_page=1" | grep -i '^x-wp-total'   # product count
curl -s   "$S/sitemap.xml" "$S/sitemap_index.xml" 2>/dev/null | grep -oE '<loc>[^<]+' | head -40
curl -s   "$S/feed" "$S/rss" "$S/blog/feed" 2>/dev/null | grep -c '<item>'  # RSS post count
curl -s   "$S" | grep -oE 'woocommerce|add-to-cart|shopify|wp-content/themes/[a-z0-9-]+'
```

`x-wp-total` is the honest count and it costs one request. Not every site is
WordPress, so also check the rendered page for a blog:

```javascript
JSON.stringify({
  blogLinks: [...document.querySelectorAll('a')]
    .map(a => a.getAttribute('href') || '')
    .filter(h => /\/(blog|news|articles|insights|posts|journal)(\/|$)/i.test(h))
    .slice(0, 10),
  feeds: [...document.querySelectorAll('link[type*="rss"],link[type*="atom"]')].map(l => l.href),
  articles: document.querySelectorAll('article').length,
  cartish: [...document.querySelectorAll('a,button')]
    .filter(e => /cart|checkout|add to (bag|cart)/i.test(e.textContent)).length
})
```

Then **report what you found before building anything**:

> The source has 88 blog posts and 412 products. Cloning the pages alone brings
> across neither. Blog: rebuild as a BrightSite blog (below). Products: see
> `commerce-catalog.md`. Which do you want in scope?

Scope is the user's call, not yours. What is not allowed is finding a blog,
saying nothing, and shipping a site with a nav link to a page that does not
exist. If the user declines, that goes in the Phase 6 report as an explicit gap.

**Count pages excluding posts and products.** A sitemap with 500 URLs is not a
500-page clone if 480 of them are posts. Posts are content, not pages, and they
come in through the blog path — never as cloned pages.

### A blog is not a set of pages — rebuild it as a BrightSite blog

This is the single most common way a clone goes wrong on a content site.

BrightSite has a real blog: one listing page, one post card design, one post
template, and posts as records. The source's blog has the same shape. So you
clone the blog **design** once — three templates — and import the posts as
posts. You do **not** create a page per post.

Doing it as pages breaks everything the blog gives you: pagination, RSS,
`BlogPosting` JSON-LD, related posts, the client's ability to write a new post
without an agency. Ninety cloned pages is not a blog; it is ninety pages that
happen to look like one, and it is unmaintainable the day after handover.

**Measure these three surfaces on the source, the same way as any section:**

| Source surface | BrightSite surface | Tools |
|---|---|---|
| The archive/index page | blog listing design | `get_blog_listing_design` → `update_blog_listing_design` → `publish_blog_listing_design` |
| One card in that index | blog post card design | `get_blog_post_card_design` → `update_blog_post_card_design` → `publish_blog_post_card_design` |
| A single post page | blog post template | `get_blog_template` → `update_blog_template` → `publish_blog_template` |
| The "keep reading" block | related posts design | `get_blog_related_posts_design` → `update_blog_related_posts_design` → `publish_blog_related_posts_design` |

Each takes `heex_staged` / `css_staged` and needs its `publish_*` call — staged
content is invisible to visitors until published, exactly like pages.

The template variables are fixed, so extract the source's card into this shape:

- **Listing** — `@posts`, `@page`, `@total_pages`. Iterate with `:for`, or call
  `component("blog-post-card", post)`.
- **Card** — `@post.title`, `.slug`, `.excerpt`, `.feature_image_url`,
  `.feature_image_id`, `.inserted_at`. Links go to `/blog/{@post.slug}`.
- **Post** — `@post.title`, `.content` (raw HTML), `.excerpt`,
  `.feature_image_url`, `.inserted_at`. `media()` and `component()` work here.

Measure the card exactly as you would any repeating item: image aspect ratio,
title size and tracking, excerpt line-clamp, meta row order (date, author, read
time), padding, gap, and the grid's column count at 1440 and 390.

**Map what you measured onto the settings, do not leave them at defaults.**
`update_blog_settings` decides what the templates are even given, and a default
that contradicts the source is a visible difference:

- `listing_url` — **match the source's path.** If the source is at `/news`,
  set `/news`, not `/blog`. Otherwise every inbound link and the old sitemap
  break.
- `posts_per_page` — count the cards on the source's first archive page.
- `show_author`, `show_published_date`, `show_read_time`, `show_tags`,
  `show_excerpts_on_listing`, `show_featured_image` — set each from what the
  source's card actually shows. A date the source hides will otherwise appear.
- `show_related_posts`, `enable_social_sharing`, `enable_search`,
  `show_sidebar` — likewise, observed not assumed.
- `blog_title`, `blog_description`, `excerpt_length`.
- `enabled: true`, and `listing_layout_id` / `post_layout_id` pointing at the
  layout you built in Phase 2 — otherwise the blog renders in the starter
  layout with none of the site's chrome.

Settings apply immediately; there is no staging flow on them.

**Getting the posts in.** Two routes, and they are not interchangeable:

- **The user has a WXR export** (`.xml` from WP Tools → Export) — hand off to
  the **migrate-wordpress-blog** skill. It parses the export, strips Gutenberg
  block comments, converts `[caption]`/`[gallery]`/`[embed]` shortcodes, and
  carries Yoast/RankMath meta across. Do not re-derive that here. It also works
  with the source already switched off, which scraping cannot.
- **No export, live site only** — read the posts from the source. Prefer
  `/wp-json/wp/v2/posts?per_page=100&page=N` (paginate until a page comes back
  empty; WordPress caps `per_page` below 100, so a short page is not the end).
  Fall back to the RSS feed, then to scraping the archive. For each post take
  the title, slug, published date, excerpt, feature image and body HTML, then
  `create_post` with `status: "draft"`.

Either way:

- **Keep the source slug exactly.** It is what is indexed.
- **Rehost every image in the body**, not just the feature image, through the
  same `request_upload` → PUT → `complete_upload` flow and the same media map.
  A body still pointing at the old domain dies when it is switched off.
- **Import as drafts**, spot-check a few, then publish. `create_post` writes
  staged fields regardless of status, so `publish_post` is required either way.
- **Redirect any path that changed** — including the archive itself if the
  source used `/news` and you could not preserve it.
- Post bodies are **raw HTML, not HEEx**. `media()` and `component()` do not
  work there; use plain `<img>` with rehosted URLs.

Finally, verify the blog like anything else: the listing at 1440 and 390 next to
the source's archive, one post next to a source post, and the card count per row
matching. A blog that was never looked at is not cloned.

### A shop is a catalog plus a storefront — split them

If fingerprinting found products, two separate jobs follow, and conflating them
is why store migrations stall:

- **The storefront's appearance** — product grid, product page, cart — is cloned
  like any other section, from this skill. Measure it, spec it, build it.
- **The catalog** — products, variants, collections, prices — is a data import
  with its own traps. Read **[commerce-catalog.md](commerce-catalog.md)** in
  this skill's directory when the user wants it in scope.

Do not clone the plumbing pages. Cart, checkout, my-account, order-received,
wishlist and compare are all handled by BrightSite's storefront templates, so
cloning them produces dead pages that shadow working ones. On most shops that
leaves three or four real content pages.

Two things to check before promising anything:

- **Is commerce enabled on the account?** If `list_products` errors, it is not,
  and that is an admin action you cannot perform. Find out now, not after the
  pages are built.
- **Is the source on a page builder?** If the Phase 1 grep found `elementor`,
  `et_pb_` (Divi) or `wpb_` (WPBakery), say so plainly: builder output is
  thousands of generated divs and inline styles, and a faithful rebuild is a
  redesign, not a migration. Quote that honestly rather than discovering it
  halfway.

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

   **What looks like one image is often several.** A hero is commonly a
   background wash, a foreground product mockup and a small overlay icon,
   stacked in one container. Missing the overlay leaves the section looking
   empty even though the background is right, and nothing about the result
   looks broken. Walk each visual container's whole subtree and count the
   layers rather than trusting the one `<img>` you noticed:

   ```javascript
   JSON.stringify([...document.querySelectorAll('SECTION img')].map(i => ({
     src: i.currentSrc || i.src,
     siblings: i.parentElement.querySelectorAll('img').length,
     pos: getComputedStyle(i).position, z: getComputedStyle(i).zIndex,
     parent: (i.parentElement.className || '').toString().slice(0, 40)
   })))
   ```

   Any entry with `siblings > 1`, or a `position: absolute` with a z-index, is
   part of a layered composition — record it in the spec as layers, back to
   front.

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

**The spec file is the source of truth, and it is a contract.** It is written
*before* the section is built, and it must be complete enough that the section
could be built from the spec alone, by someone who never saw the page. If you
build straight from a browser session, you are building from whatever you happen
to remember, and you will fill the gaps by guessing — which is indistinguishable
from a careful build until Phase 5, or until the client finds it.

That "someone who never saw the page" is literal when the work is delegated. If
you hand a section to a subagent, the spec's contents go **inline in its
prompt** — never "read the spec file" or "see the tokens doc". A builder that
has to go and look something up will instead approximate it. The file still gets
written: it is the audit trail when a section turns out wrong, and it is what
you re-read in Phase 5 to decide whether the extraction or the build was at
fault.

Dispatch follows the same split as the complexity budget: a simple section is
one unit of work; a section with three card variants is one unit per variant
plus the wrapper, with the variants done before the wrapper that embeds them.
Never bundle unrelated sections — a CTA and a footer are different designs and
handing them over together guarantees one of them gets the less careful pass.

### Enumerate every element in the section — do not extract by tag list

The fastest way to lose content is to extract "the headings and paragraphs" and
never notice what else was there. A section rebuilt from `h2`/`h3`/`p` silently
drops icons, italic subtitles, dividers, badges and buttons, and the result
looks tidy enough that nothing flags it.

Walk the section and list **every** element that renders something, then account
for each one:

```javascript
(function (sel) {
  const root = document.querySelector(sel), out = [];
  root.querySelectorAll("*").forEach(e => {
    const r = e.getBoundingClientRect();
    if (r.width < 2 || r.height < 2) return;
    const own = [...e.childNodes]
      .filter(n => n.nodeType === 3 && n.textContent.trim()).map(n => n.textContent.trim()).join(" ");
    const cs = getComputedStyle(e), before = getComputedStyle(e, "::before").content;
    if (own || e.tagName === "IMG" || e.tagName === "SVG" ||
        (before && before !== "none" && before !== '""'))
      out.push({tag: e.tagName, cls: (e.className || "").toString().slice(0, 40),
                text: own.slice(0, 60), pseudo: before,
                font: cs.fontFamily.split(",")[0], size: cs.fontSize, style: cs.fontStyle});
  });
  return JSON.stringify(out, null, 1);
})('SECTION_SELECTOR')
```

Two things this catches that a tag-list extraction cannot:

- **Icon fonts.** An icon may be a `<span>` whose glyph comes from a
  `::before` on a webfont (ElegantIcons, FontAwesome). It has no text, no
  `<img>` and no `<svg>` — it is invisible to any selector you would think to
  write. The `pseudo` field above is what reveals it. Reproduce it as inline
  SVG at the measured size and colour; do not try to load the icon font.
- **Text in unexpected elements.** A subtitle may be a bare `<div>`, not a
  `<p>`. Its font is worth reading on its own: a serif italic line inside an
  otherwise sans-serif section is a deliberate choice, and it is the sort of
  detail that makes a clone feel right.

### Measure the container before anything inside it

Extract the section's **layout container** first — its width, max-width, padding
and left/right edges — and match that before touching any child:

```javascript
(function (sel) {
  const el = document.querySelector(sel), r = el.getBoundingClientRect(),
        c = getComputedStyle(el);
  return JSON.stringify({left: Math.round(r.left), right: Math.round(r.right),
    width: Math.round(r.width), maxWidth: c.maxWidth, padding: c.padding,
    margin: c.margin, display: c.display, gap: c.gap, justifyContent: c.justifyContent});
})('SELECTOR')
```

The container is what positions everything inside it. A 1170px centred container
puts a logo at x=170 by itself; a full-bleed one puts it at x=30 no matter what
you do to the logo.

**Never force a child's position to hit a measured number.** If a child is in
the wrong place, the container or the spacing between siblings is wrong — fix
that. Overriding a container's `max-width` to make three x-coordinates line up
produces a nav whose numbers match and whose overall width is visibly wrong,
because the numbers you checked were the only ones you looked at.

Measure, in this order, and match each before moving down:

1. container edges, width, max-width, padding
2. spacing between siblings (flex `gap`, per-item padding/margin)
3. individual element positions — which should now already be right

Also measure the **span** of any repeating row (first item's left edge to last
item's right edge). A menu whose endpoints match but whose span does not has the
wrong inter-item spacing, and that is invisible in an endpoint check.

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

Copy the text content verbatim. "SHINE BRIGHT" is not "We Shine Bright".

### Capture every state as a diff, not as prose

For anything with more than one state (a scrolled header, a hovered card, a
non-active tab), capture **both** states and record the difference plus the
transition. "It gets a bit darker on hover" is not a spec. `backgroundColor:
rgb(255,255,255) → rgb(244,244,245), transition: background-color 0.2s ease` is.

Snapshot the same properties before and after, then diff them mechanically —
comparing two dumps by eye is how a changed `letter-spacing` gets missed.

**The harness.** Run this once to define it, then call `__snap()` before and
after each trigger and `__diff()` to compare:

```javascript
window.__snap = function (sel) {
  const el = document.querySelector(sel); if (!el) return null;
  const cs = getComputedStyle(el), r = el.getBoundingClientRect(), o = {};
  ['color','backgroundColor','backgroundImage','borderColor','borderRadius','borderWidth',
   'boxShadow','opacity','transform','filter','backdropFilter','fontSize','fontWeight',
   'letterSpacing','textDecorationLine','padding','margin','height','width','position',
   'top','left','zIndex','visibility','display','transition','scale','translate']
    .forEach(k => o[k] = cs[k]);
  o._rect = {t: Math.round(r.top), l: Math.round(r.left),
             w: Math.round(r.width), h: Math.round(r.height)};
  return o;
};
window.__diff = function (a, b) {
  const out = {};
  Object.keys(a).forEach(k => {
    const x = JSON.stringify(a[k]), y = JSON.stringify(b[k]);
    if (x !== y) out[k] = a[k] + "  →  " + b[k];
  });
  return JSON.stringify(out, null, 1);
};
```

**Hover.** A real pointer hover is what triggers `:hover`; dispatching a
`mouseover` event does not, and CSS-only hovers will read as unchanged. Move the
actual pointer to the element's centre, then snapshot:

```bash
agent-browser eval 'window.__before = __snap("SELECTOR"), "ok"'
agent-browser hover "SELECTOR"      # a real pointer move
agent-browser eval '__diff(window.__before, __snap("SELECTOR"))'
```

If the tool cannot hover, read the `:hover` rule out of the stylesheet instead
of guessing — and say in the spec that it came from the CSS, not from a measured
state:

```javascript
JSON.stringify([...document.styleSheets].flatMap(sh => {
  try { return [...sh.cssRules]; } catch (e) { return []; }   // cross-origin
}).filter(r => r.selectorText && r.selectorText.includes(':hover'))
  .map(r => r.selectorText + " { " + r.style.cssText + " }").slice(0, 60))
```

Hover the **element that carries the rule**, not the one that changes: a card
whose image scales usually has the transition on the image but `:hover` on the
card. Check the parent when a hover reads as no-change.

**Scroll.** Snapshot at 0, scroll past the trigger, snapshot again:

```bash
agent-browser eval 'window.scrollTo(0,0); window.__a = __snap("header"), "ok"'
agent-browser scroll down 400
agent-browser eval '__diff(window.__a, __snap("header"))'
```

Then find the **threshold**, because "changes on scroll" does not tell you where.
Step down the page and record the first offset at which the value flips:

```javascript
(async function (sel, prop) {
  const out = []; let last = null;
  for (let y = 0; y <= 600; y += 25) {
    window.scrollTo(0, y);
    await new Promise(r => requestAnimationFrame(() => setTimeout(r, 60)));
    const v = getComputedStyle(document.querySelector(sel))[prop];
    if (v !== last) { out.push({y, [prop]: v}); last = v; }
  }
  window.scrollTo(0, 0);
  return JSON.stringify(out);
})('header', 'backgroundColor')
```

The first entry after y=0 is your threshold. A header that flips at 50px and one
that flips at 300px are different builds.

**Tabs and carousels.** Click each control, snapshot the panel, and record which
content belongs to which state. For a carousel, snapshot the same node several
seconds apart *without touching anything* — if it changed, it is time-driven and
you also need the interval.

Record every diff in the spec's States & behaviours block: trigger, state A,
state B, transition, and where the transition is declared.

### Write the spec

One file per section, before building:

```markdown
# <Section> spec
- Source URL / selector:
- Screenshot (source):
- INTERACTION MODEL: static | hover | click-driven | scroll-driven | time-driven
  (say the mechanism, not just the category: "scroll-driven,
  IntersectionObserver at ~30%", "time-driven, auto-advances every 5s")
- Becomes: inline page section | BrightSite component (slug)
- Target: page slug / component slug
- Media: video | Lottie | canvas | still images  (checked, not assumed)

## Structure
<what contains what>

## Element inventory
Every element the section renders, from the sweep — including icon-font
`::before` glyphs, dividers, badges and bare `<div>` subtitles. Each one is
either built or explicitly listed as dropped, with a reason.

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

## Per-state content (tabs, carousels, accordions — otherwise N/A)
### State: "<tab label>"
- Heading: …
- Cards: [{title, body, image, link}, …]
### State: "<next tab label>"
- …

## Assets
source URL → media id
Layered compositions listed as layers, back to front (a background wash, a
foreground mockup and an overlay icon are three assets, not one).

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
- [ ] The spec is self-contained — buildable without going back to the browser
- [ ] The spec is under ~150 lines; if not, the section is split
- [ ] Every CSS value came from `getComputedStyle()`, none estimated
- [ ] The interaction model is identified by scrolling and waiting *first*, and
      written into the spec with its mechanism
- [ ] The section was checked for `<video>` / Lottie / `<canvas>` before any
      HTML mockup of its contents was planned
- [ ] Heights checked at two viewport heights — is it px or `100vh`?
- [ ] The section's descendants were swept for `position: fixed` / `sticky`
- [ ] For stateful elements: both states captured **as a property diff**, with
      the transition and where it is declared
- [ ] Hover states captured with a real pointer hover (or read from the
      stylesheet and labelled as such), not assumed
- [ ] Scroll-triggered elements have a measured threshold, not "on scroll"
- [ ] For tabs/carousels: every state's content captured, not just the default;
      carousel interval measured by waiting
- [ ] Every image found, including background images, overlays and layers
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
- [ ] Screenshot the section on **both** sides and read **both** images
- [ ] Section height matches the source's (a taller section means spacing drift)
- [ ] Text wraps to the same number of lines — same width and same wrap means
      the type metrics match; a different wrap at the same width means they do not
- [ ] **Every behaviour in the spec was triggered on the built section and
      diffed against the source's** — hover each interactive element, scroll
      past each threshold, click each tab, wait out each carousel. A section
      whose states were never fired has not been verified, only looked at.
- [ ] The interaction model matches: a scroll-driven section changes on scroll
      without a click, a time-driven one changes on its own

That fourth box is the one that keeps being missed. Reorganising the source's
content because the new arrangement seems tidier is not cloning it.

## Phase 4: Assembly

Create or update each page (`create_page` / `update_page`) at the slug from the
output plan, assign the layout, then `publish_page`. Content only reaches the
live site on publish.

Then wire the page-level behaviour that is not any one section's: scroll snap on
the page container, a pinned backdrop's stacking context, smooth scroll, and
anything an IntersectionObserver drives across sections.

Before finishing:

- [ ] Every page in the output plan exists at its planned slug
- [ ] Every page that existed **before this run** and is not a starter
      placeholder is still there and unchanged
- [ ] Starter template pages that are not part of the source site are deleted —
      and nothing else is
- [ ] Redirects exist for any source path whose slug had to change
- [ ] Internal links point at page ids (`page_url("id")`), not at the source
      domain or at hardcoded paths

If a page you planned to create already existed and you did not agree a
replacement, stop and ask rather than overwriting it.

## Phase 5: Visual QA — mandatory

**Every check in this phase compares two sides.** Run each one against the
source *and* the clone and diff the results. A single-sided check — "the nav
rendered", "no broken images" — tells you the page is not crashing, which is not
the question.

**Looking at your own output is not a comparison.** Cropping your page,
inspecting it, and judging it "looks right" is the single most common way a
clone ships wrong — you are checking it against your memory of the source, and
your memory is of what you built. The two images must be in front of you, one
after the other, at the same viewport and the same crop.

The procedure, exactly:

```bash
agent-browser set viewport 1440 900
agent-browser open "<source-url>";     agent-browser screenshot /abs/shots/cmp_src.png  --full
agent-browser open "<brightsite-url>"; agent-browser screenshot /abs/shots/cmp_mine.png --full
# crop the SAME y-range out of both, then read both files
```

Then read **both** crops. If you only read one, you have not compared anything.

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

### Behaviour parity — the states, not just the pixels

Everything above compares two **static** pages. A clone can pass every one of
those checks and still be dead: the header never darkens, the cards do not lift,
the carousel sits on slide one forever. Screenshots cannot fail on any of that,
which is exactly why it ships broken.

So run the Phase 3 capture harness a second time — on the source **and** on the
clone — and compare the diffs. The target is not "something changed on hover";
it is "the same properties change, by the same amounts, over the same duration".

Define `__snap`/`__diff` on both pages, then for each behaviour in the spec:

```bash
# hover, on each side in turn
agent-browser open "<url>"
agent-browser eval 'window.__a = __snap("SELECTOR"), "ok"'
agent-browser hover "SELECTOR"
agent-browser eval '__diff(window.__a, __snap("SELECTOR"))'
```

Compare the two outputs property by property:

| Symptom | What it means |
|---|---|
| clone diff is `{}`, source's is not | the state was never built |
| same properties, different values | built from memory, not from the spec |
| extra properties in the clone | you invented a behaviour the source has not |
| values match, `transition` differs | right end state, wrong feel — still wrong |
| source diff is `{}`, clone's is not | you added a hover the source does not have |

A clone that animates *more* than the source is as wrong as one that animates
less. An added hover lift is not a free improvement; it is a difference.

Run this for every entry in every spec's States & behaviours block:

- [ ] **Hover** — each interactive element: nav links, buttons, cards, images.
      Same properties, same values, same transition.
- [ ] **Scroll state** — the header (and anything else scroll-triggered)
      compared at scroll 0 and past the trigger, on both sides.
- [ ] **The threshold matches** — run the stepped-scroll probe on both. A header
      that flips at 50px on the source and 300px on the clone is not a match,
      and both look correct in any single screenshot.
- [ ] **Tabs** — click each one on both sides; the panel content and the
      switching transition match.
- [ ] **Carousel** — wait the same number of seconds on both without touching
      anything. Same advance interval, same number of slides. A static clone of
      a rotating hero passes every pixel check ever written.
- [ ] **Scroll-driven switching** — if the source's sidebar or tab indicator
      changes by itself as content passes, confirm the clone's does too, at the
      same points. Rebuilding it as click-driven is the failure this catches.
- [ ] **Smooth scroll** — if the source runs Lenis or Locomotive, confirm the
      clone does. Native scrolling next to eased scrolling is immediately
      obvious to anyone who used the old site.
- [ ] **Focus states** — tab through the page. Removing a focus ring the source
      had is an accessibility regression, not a styling choice.
- [ ] **Reduced motion** — if the source honours
      `prefers-reduced-motion`, the clone must too.

Do this at 1440 and again at 390: a hover that works on desktop can be attached
to a control that does not exist on mobile, and a sticky header often behaves
differently at the small breakpoint.

**Interaction-model regression check.** Before signing off any section the spec
marked scroll-driven or time-driven, confirm the clone is not click-driven.
Scroll or wait — do not click — and verify it changes on its own:

```javascript
// scroll-driven: the panel should change without any click
(async function (sel) {
  const before = document.querySelector(sel).textContent.trim().slice(0, 60);
  window.scrollBy(0, 600);
  await new Promise(r => setTimeout(r, 800));
  const after = document.querySelector(sel).textContent.trim().slice(0, 60);
  return JSON.stringify({before, after, changedOnScroll: before !== after});
})('PANEL_SELECTOR')
```

`changedOnScroll: false` on the clone where the source says `true` means the
section was built with the wrong interaction model. That is a rebuild of the
section, not a CSS fix, and it is far cheaper to find here than after handover.

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

- Source URL → BrightSite page (and slug) for each page
- Pages that existed before this run, and confirmation they were preserved;
  any replacement that was explicitly agreed
- **What the source contained vs what was brought across** — the fingerprint
  counts (pages, posts, products) next to what was actually migrated. A source
  with 88 posts and a clone with 0 is a stated gap, never an unmentioned one.
- Blog: listing/card/post/related designs published, settings mapped, posts
  imported (and how many are still draft), `listing_url` preserved or redirected
- Catalog: products imported and how many are still draft, collections
  activated, variable products flagged; what did not migrate (customer accounts,
  orders, reviews, bookings, memberships)
- Sections built, and which became components vs inline
- Spec files written — the count should equal the number of sections built. A
  section built without a spec is the one to re-check first.
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
- **Do not force a child's position to match a number.** Positions come from
  the container and the spacing; overriding a child hides the real difference
  and breaks the overall proportions.
- **Do not "verify" by looking at your own output alone.** Reading one crop and
  calling it a match is not a comparison; it is a recollection. Read the source
  crop and yours, in the same turn.
- **Do not stop at the values you set.** A paragraph rendering at 15px when the
  spec says 16px means an earlier rule is winning. Read the *computed* value
  back, not the declaration you wrote.
- **Do not extract by tag list.** `h2`/`h3`/`p` silently drops icons,
  subtitles, dividers and buttons. Enumerate everything the section renders.
- **Do not assume a font seen in the stylesheet is used where you expect.** A
  display serif may appear on exactly one italic subtitle and nowhere else.
- **Do not check only endpoints.** Matching the first and last element while the
  span between them differs means the spacing is wrong.
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
- **Do not build a section from a browser session instead of a spec.** The spec
  is written first, and it is what makes the build reproducible and auditable.
- **Do not hand a builder a pointer instead of the content.** "See the spec
  file" or "use the colours in the tokens doc" produces approximation; the
  values go inline in the prompt.
- **Do not bundle unrelated sections into one unit of work.** A CTA and a
  footer are separate designs; together, one of them gets the sloppier pass.
- **Do not let one section's spec grow past ~150 lines.** That is the signal to
  split it, not to push through.
- **Do not build an HTML mockup of something that is a video, Lottie or
  canvas.** Check the tag names before designing the rebuild.
- **Do not extract only the state that happened to load.** Click every tab,
  wait out every carousel, and record the content of each state.
- **Do not overwrite pages you did not create in this run.** Inventory the
  account first, plan the slugs, and ask before replacing anything.
- **Do not flatten the source's URLs.** `/about-us/` stays `about-us`; a
  "tidier" slug breaks every inbound link unless you also create the redirect.
- **Do not verify a clone with screenshots alone.** Static captures cannot fail
  on a dead hover, a header that never darkens or a carousel stuck on slide one.
  Fire every state on both sides and diff them.
- **Do not describe a state instead of measuring it.** "Darkens slightly on
  hover" is not a spec; a property diff with values and a transition is.
- **Do not fake a hover with a dispatched `mouseover`.** CSS `:hover` needs a
  real pointer; a synthetic event reads as no-change and you will conclude the
  element has no hover state.
- **Do not accept "it changes on scroll" without the threshold.** 50px and
  300px look identical in any one screenshot and feel completely different.
- **Do not add a behaviour the source lacks.** An extra hover lift or fade-in is
  a difference, not an improvement — the clone should animate exactly as much as
  the original, no more.
- **Do not ship a static clone of a rotating hero.** Wait several seconds on
  both sides before calling a hero static.
- **Do not clone a blog as a page per post.** It looks identical on day one and
  throws away pagination, RSS, `BlogPosting` JSON-LD, related posts and the
  client's ability to publish. Clone the three blog designs; import posts as
  posts.
- **Do not leave the blog settings at their defaults.** `posts_per_page`,
  `show_author`, `show_read_time` and the rest are visible differences, and
  `listing_url` defaulting to `/blog` breaks a source that lived at `/news`.
- **Do not count posts and products as pages.** A 500-URL sitemap where 480 are
  posts is a 20-page clone, and quoting it as 500 misprices the job.
- **Do not clone the storefront plumbing pages.** Cart, checkout and account are
  BrightSite templates; cloned copies are dead pages shadowing working ones.
- **Do not silently skip content the source has.** Finding a blog or a shop and
  saying nothing is the failure this whole fingerprinting step exists to stop.
  Report it, let the user set scope, and record the decision.
