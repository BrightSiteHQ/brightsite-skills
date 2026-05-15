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

Return the post ID and a note like:

> Draft created: post ID `8fd2k3jq91pn`. Open it in the BrightSite editor to add a feature image, internal links, and publish when ready.

## Anti-patterns to avoid

- **Don't auto-publish.** Even if the user says "and publish it" — push back once. Tell them: "I always create as draft so you can review in the editor. Once you've previewed it, you can publish with one click."
- **Don't generate posts without an outline.** If the user gives just a topic, ask for an outline first. Don't produce a generic post.
- **Don't pad to hit word count.** If the natural length is 700 words and the target was 1000, return 700 words with a note. Padded posts hurt SEO and bounce rate.
- **Don't use AI tells.** Strip any of: "dive into," "unleash," "in today's [adjective] world," "in conclusion," "It's important to note that," em-dash-heavy sentences, "let's explore." If the user wants those, they can add them back.
- **Don't fabricate stats or quotes.** If the outline mentions "stat about adoption rates," ask the user for the source. Don't make it up.

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

- `mcp__brightsite__list_posts` — to find internal link opportunities
- `mcp__brightsite__list_pages` — same
- `mcp__brightsite__get_blog_settings` — to confirm blog URL prefix for internal links
