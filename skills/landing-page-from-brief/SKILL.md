---
name: landing-page-from-brief
description: Build one campaign or offer landing page on a BrightSite site from a brief — intake, page structure, conversion copy, index/noindex decision, then a draft page built section by section in the site's existing design. Use when the user asks for a "landing page," "campaign page," "ad landing page," "offer page," "waitlist page," "lead-gen page," or "X vs Y comparison page." For many near-identical geo pages use generate-location-pages instead. For restyling a whole site use staging-redesign.
---

# Landing Page from Brief

A landing page is not a homepage. A homepage serves several intents; a landing page serves one: **one offer, one audience, one primary action.**

## When to use this

- A client is running ads, an email campaign, or a launch and needs a page for that one offer.
- The client wants a comparison or "alternative to X" page for search.
- A waitlist, download, or lead-gen page.

Do NOT use this for:
- Dozens of pages that differ only by location — use `generate-location-pages`.
- Restyling the whole site — use `staging-redesign`.

## Inputs you need from the user

Ask in one batch, and only for what's missing. If the user can't answer something, state one assumption in one line and continue.

1. **Account ID**, plus a layout ID if the page shouldn't use the default layout.
2. **The one primary action** (trial, demo, call, buy, waitlist, download) and what counts as a conversion. If it's a form submission, which BrightSite form to use (`list_forms`).
3. **The offer** — exactly what the visitor gets.
4. **Audience** — who they are, the problem they're solving, and their top 3 objections.
5. **Traffic source** (ads, search, social, email) and what visitors already know. For ads, the ad headline.
6. **Proof** — logos, testimonials, numbers, case studies, screenshots or video. Guarantees and refund or cancellation terms.
7. **Brand voice.**

Never invent testimonials, stats, client names or guarantees. If proof is missing, leave that section out and tell the user.

## Workflow

### Step 1: Pick a layout type

Pick one and give the reason in one line:

| Type | Use when |
|---|---|
| **A. Classic hero plus sections** | Default. The offer is clear from a hero visual. |
| **B. Long-form story** | Skeptical audience that needs educating. |
| **C. Minimal conversion page** | High-intent traffic (email to known users) or a short offer like a download or waitlist. |
| **D. Comparison page** | Search intent like "X vs Y" or "alternative to X". |

### Step 2: Outline and copy before any HEEx

Return these in order and wait for approval:

1. **Section outline.** Default structure:
   - Above the fold: headline (outcome + audience), subheadline (how, specific), primary CTA, one proof signal, hero visual.
   - Mid page: problem to solution, 3-5 outcome-led benefits, how it works in 3 steps, social proof.
   - Bottom: FAQ with 6-12 questions, risk reversal, final CTA identical to the top one.
   - Type C trims the mid page.
2. **Hero copy** — headline, subheadline, CTA label, proof line.
3. **Benefits, how it works, FAQ** questions and answers.
4. **Index decision** plus `meta_title` and `meta_description` (Step 4).

### Copy rules

- Headline formulas: "{Outcome} without {pain}", "The {category} for {audience}", "{Result} in {time}".
- For ad traffic, mirror the ad's headline and promise in the hero.
- CTA = verb + what they get ("Book a free estimate", "Get the checklist"). Never "Learn more" or "Submit". One primary CTA above the fold, with no competing CTA.
- Benefit before feature. Benefit bullets: bold benefit, then the proof or detail.
- Be specific. Replace "save time" and "streamline" with a measurable outcome from the client's real data ("Cut weekly reporting from 4 hours to 15 minutes").
- At least one risk reversal the client actually offers (free trial, no credit card, cancel anytime, money-back, free estimate).
- Put proof next to the claim it supports, not only at the bottom. For high-friction offers, move the FAQ higher.
- Banned filler: "Elevate", "Seamless", "Unleash", "Next-gen", "Game changer", "Delve", "Tapestry", "In the world of". No Lorem Ipsum, placeholder names or brands, or made-up round numbers.
- Sentence-case headings. Active voice. No exclamation marks in form success messages. Error messages say what happened and what to do.

### Step 3: Build section by section

Load the **visual-editor-authoring** skill first and follow it (edit markers, `props_schema`).

- Build in this order: hero, benefits, how it works, proof, FAQ, final CTA. Show each section before starting the next.
- Never regenerate the whole page to change one section.
- Call `mcp__brightsite__create_page` once with `status: "draft"` (check `list_pages` first so the slug doesn't collide), then use `mcp__brightsite__update_page` for later sections.
- Visual values come from the site's own design. Read `mcp__brightsite__get_global_code` (Tailwind config, CSS) and reuse existing layouts and components. Don't introduce new fonts, colors or spacing values.
- Keep any motion short and purposeful: transform/opacity only, under about 300ms, and respect `prefers-reduced-motion`.
- Render the header and footer through the layout builtins, never hand-built.

### Step 4: Index decision

- **Ad-only or time-bound offers:** set `meta_robots: "noindex"`, and don't add the page to the builtin navigation.
- **Evergreen offers:** keep the page indexable. Write a `meta_title` under 60 characters and a `meta_description` under 155. Link to it from relevant existing pages (ask before editing them). Keep the FAQ as plain question-and-answer text.
- `meta_*` fields apply live immediately (not staged). Confirm before setting them on an existing page.

### Step 5: Hand back

- Return the draft `path` from `create_page`.
- Ask the user to check the form (it posts, the success message reads right) and the hover, focus and pressed states on the CTAs.
- Tell them to publish with `publish_page` after review.
- If the page is indexable, suggest running `bulk-seo-audit`.

## Anti-patterns to avoid

- **Writing HEEx before the outline and copy are approved.** Copy changes are cheap in an outline and expensive in markup.
- **Two primary CTAs above the fold.** One action per page.
- **Fabricated proof.** No invented testimonials, logos, stats or guarantees.
- **Leaving an ad-only page indexable.** It competes with the site's real pages and goes stale when the campaign ends.
- **Importing a new visual style** instead of the site's tokens.
- **Publishing from this skill.** Create a draft; the user publishes after review.

## Example invocation

> Account `YOUR_ACCOUNT_ID`. Google Ads campaign for a free furnace tune-up in October. Ad headline: "Free furnace tune-up before winter." Goal: book a visit through the "Service request" form. Proof: 4.8 stars from 312 Google reviews, 22 years in business. Guarantee: no charge if the tune-up finds nothing to fix.

## Tools used

- `mcp__brightsite__get_global_code` — site tokens (Tailwind config, CSS)
- `mcp__brightsite__list_layouts` / `mcp__brightsite__list_components` — reuse existing layouts and components
- `mcp__brightsite__list_forms` — pick the conversion form
- `mcp__brightsite__list_pages` — slug collision check
- `mcp__brightsite__create_page` / `mcp__brightsite__update_page` / `mcp__brightsite__get_page` — build the draft
- `mcp__brightsite__publish_page` — only when the user asks
