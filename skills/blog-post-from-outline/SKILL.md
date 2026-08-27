---
name: blog-post-from-outline
description: Turn an outline (bullet points, headings, or a rough brief) into a full BrightSite blog post draft with proper structure, meta title, meta description, and excerpt. Always creates as `draft` — never publishes. Use when the user mentions "draft a blog post," "write a post from this outline," "turn this into a blog post," "blog draft," "post from brief," or has a structured outline they want fleshed out.
---

# Blog Post from Outline

Take a structured outline and produce a publishable-quality draft post in BrightSite. The agency reviews and edits before publishing.

This skill is intentionally NOT a "generate me a blog post from a topic" prompt. That produces generic, low-value content. This skill requires the user to have already done the thinking — outline, key points, target keyword — and turns that thinking into a clean draft.

## When to use this

- Agency has a content brief from a client and needs a fast draft.
- Subject matter expert recorded a voice memo / Loom and the agency has a transcript outline.
- Writer has a structured outline but doesn't want to write the connective prose.

Do NOT use this for:
- Generating posts from just a title or topic (output will be generic).
- Long-form pillar content (>2000 words) — the model produces better results with more iteration; do it in pieces.

## Inputs you need from the user

1. **Account ID**
2. **Outline** — bullet points, H2/H3 headings, key claims. The more structured, the better.
3. **Target keyword** — the primary phrase the post should rank for.
4. **Audience** — one sentence on who this is written for (e.g. "small business owners evaluating CRMs," "homeowners considering solar").
5. **Voice/tone reference** — link to or paste an existing post the client has written, OR describe the tone ("conversational, no fluff, US English").
6. **Word count target** — default 800-1200 words.
7. **CTA** — what action should the reader take at the end?

## Workflow

### Step 1: Confirm understanding

Before drafting, parrot back to the user:
- The H2 structure you'll use
- The target keyword and where you'll place it (title, H1, first 100 words, one H2, meta)
- The CTA
- The voice you're targeting

Wait for confirmation. Don't draft if any of those are unclear — go back and ask.

### Step 2: Draft the post

Write the post following these constraints:

- **Title**: under 60 chars, includes the target keyword naturally, action/benefit-oriented (not "How to do X" if there's a better angle).
- **First paragraph**: hook the reader in 2-3 sentences, mention the target keyword once, set up what the rest of the post delivers. No "In today's fast-paced world" openers.
- **H2 structure**: matches the outline. Each H2 should contain at least one tangible takeaway, example, or piece of evidence — not just transitional copy.
- **Voice**: match the reference. If no reference, default to: short sentences, active voice, no buzzwords (utilize, leverage, synergy), no AI tells ("dive in," "unleash," "in conclusion").
- **CTA**: one paragraph at the end. Direct ask, not a soft "let us know what you think."
- **Internal links**: if the user mentioned related posts or pages, work in 1-2 natural internal links.
- **Length**: hit the word count target ±15%.

### Step 3: Generate the metadata

- `meta_title` — under 60 chars. Often the same as the post title, but can be tighter for search.
- `meta_description` — 130-155 chars. Includes the target keyword. Reads like a benefit, not a summary.
- `excerpt` — 1-2 sentences, used on the blog listing page. Should make someone want to click. Different from meta_description (meta is for Google, excerpt is for humans browsing your blog).
- `slug` — kebab-case version of the title, but trim stop words. Under 60 chars.

### Step 4: Show the user the draft before creating

Output the full post in markdown to the user with a clear delineation:

```
--- DRAFT PREVIEW ---
Title: ...
Slug: ...
Meta title: ...
Meta description: ...
Excerpt: ...

[Full post body in markdown]
--- END PREVIEW ---
```

Ask the user to approve or request changes. Common revisions: tone is off, one H2 is weak, CTA needs work. Iterate before creating in BrightSite.

### Step 5: Create the post (draft only)

Once approved, convert the markdown body to clean HTML. Post bodies are plain HTML only — they do NOT support HEEx, Phoenix template helpers, or page components. If the outline requires dynamic/templated content, that belongs on a page (use a different skill), not a post.

Call `mcp__brightsite__create_post` with:

- `account_id`
- `title`
- `slug`
- `content` — HTML body
- `excerpt`
- `meta_title`
- `meta_description`
- `status: "draft"` — never "published" from this skill. Note: even `status: "published"` writes content into staged fields; going live requires a separate `mcp__brightsite__publish_post` call. Always create as draft so the user explicitly publishes after review.
- `feature_image_id` or `feature_image_url` — if the user provided one; otherwise skip

Optional params worth setting when the user gave you the information. Skip any the user didn't ask for — don't invent values:

- `related_post_ids` — array of post IDs for the manually-curated "related posts" block at the bottom of the post, in display order. See "Related posts" below.
- `published_at` — ISO 8601 datetime (e.g. `2026-01-15T12:00:00Z`) controlling the post date shown publicly. Only set this if the user asked for a specific date (backdating a migrated post, scheduling ahead). Otherwise leave it off — it defaults to the moment the post is first published.
- `canonical_url` — set when the post is republished from somewhere else and the original should get the SEO credit. Don't set it for original content.
- `meta_robots` — e.g. `noindex` or `noindex nofollow`. Only for posts that shouldn't rank (thank-you pages, gated content, temporary posts).
- `og_image_id` — media file ID for the social share image, when it should differ from the feature image.
- `structured_data` — per-post schema.org JSON-LD, emitted server-side into `<head>`. Do NOT put a `<script type="application/ld+json">` tag in the post body — it duplicates on every LiveView navigation. BlogPosting, Organization/LocalBusiness, WebSite, and BreadcrumbList are already emitted automatically, so only add post-specific entries. Shape: `{"faq": [{"question": "...", "answer": "..."}], "product": {...}, "raw": "<JSON-LD string>"}`. Prefer the typed `faq`/`product` keys over `raw` — they validate. There is no `article` key; the post already emits BlogPosting.

Return the post ID and a note like:

> Draft created: post ID `8fd2k3jq91pn`. Open it in the BrightSite editor to add a feature image, internal links, and publish when ready.

### Related posts

Two different things link a post to other posts. Don't confuse them:

1. **Inline internal links** — `<a href>` tags you write into the post body (Step 2). These are the ones that pass SEO value and give readers context mid-article.
2. **The related posts block** — a "Keep reading" section BrightSite renders at the bottom of the post. Controlled by `related_post_ids`.

To set the block, pass `related_post_ids` on `create_post` or `update_post` — an array of post IDs in the order they should display. On `update_post` it replaces the whole selection; pass `[]` to clear it.

Two things to know before you use it:

- **It requires the `show_related_posts` blog setting to be on.** If that setting is off, the IDs save but nothing renders on the public site. Check with `mcp__brightsite__get_blog_settings` first. If it's off and the user wants the block, turn it on with `mcp__brightsite__update_blog_settings` (`show_related_posts: true`) — but tell them first, because it's a blog-wide change that affects every post, not just this one.
- **It is not staged.** Unlike `content_staged` and `excerpt_staged`, `related_post_ids` takes effect immediately — no `publish_post` call needed. So setting it on a draft changes what other, already-live posts can point at right away. Say so when you set it.

Pick related posts from `mcp__brightsite__list_posts`. Choose 2-4 posts that genuinely continue the reader's thought, and stick to ones with `status: published` — a draft has no public URL to link to.

## Anti-patterns to avoid

- **Don't auto-publish.** Even if the user says "and publish it" — push back once. Tell them: "I always create as draft so you can review in the editor. Once you've previewed it, you can publish with one click."
- **Don't generate posts without an outline.** If the user gives just a topic, ask for an outline first. Don't produce a generic post.
- **Don't pad to hit word count.** If the natural length is 700 words and the target was 1000, return 700 words with a note. Padded posts hurt SEO and bounce rate.
- **Don't use AI tells.** Strip any of: "dive into," "unleash," "in today's [adjective] world," "in conclusion," "It's important to note that," em-dash-heavy sentences, "let's explore." If the user wants those, they can add them back.
- **Don't fabricate stats or quotes.** If the outline mentions "stat about adoption rates," ask the user for the source. Don't make it up.
- **Don't flip blog-wide settings silently.** `show_related_posts` is one switch for the entire blog. If you turn it on so this post's related block renders, say so — the user may not want it on their other 40 posts.
- **Don't set `related_post_ids` to fill the block.** If nothing on the site is genuinely related, leave it empty. A "keep reading" block full of unrelated posts is worse than no block.

## Example invocation

> Account `YOUR_ACCOUNT_ID`. Target keyword: "small business HVAC marketing." Audience: HVAC company owners with 5-20 employees. Voice: like the post at /blog/hvac-seo-basics on the same site. 1000 words. CTA: book a strategy call.
>
> Outline:
> - Most HVAC marketing fails because it's interchangeable
> - 3 things that actually work: Google Business Profile reviews, neighborhood-specific landing pages, seasonal email follow-up
> - For each: why it works, one example, how to start this week
> - Wrap with: pick one, do it for 90 days

## Tools used

- `mcp__brightsite__create_post` — create the draft

(Optional, if you want to enrich the draft)

- `mcp__brightsite__list_posts` — to find internal link opportunities, and to pick IDs for `related_post_ids`
- `mcp__brightsite__list_pages` — same
- `mcp__brightsite__get_blog_settings` — to confirm the blog URL prefix for internal links, and to check whether `show_related_posts` is on
- `mcp__brightsite__update_blog_settings` — only to turn `show_related_posts` on, and only after telling the user it's a blog-wide change
- `mcp__brightsite__list_media` — to find an image ID for `og_image_id`
