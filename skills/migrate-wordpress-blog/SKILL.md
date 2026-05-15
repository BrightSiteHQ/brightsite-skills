---
name: migrate-wordpress-blog
description: Import posts from a WordPress XML export (WXR) into BrightSite. Preserves slugs, converts WP shortcodes/blocks to clean HTML, maps feature images, and generates 301 redirects for any slug that changes. Use when the user mentions "migrate WordPress," "WordPress export," "WXR file," "import from WP," "move blog from WordPress," or has a `.xml` file from a WordPress site.
---

# Migrate WordPress Blog

Take a WordPress WXR export file and import the blog into BrightSite. The goal: preserve as much SEO equity as possible (slugs, internal links, image references) and never break a URL that ranks.

## When to use this

- Client is moving off WordPress to BrightSite.
- Agency has a WXR export (`.xml` file from WP Tools → Export → Posts).
- Could be 10 posts or 10,000 — the workflow is the same.

## Inputs you need from the user

1. **Account ID** — destination BrightSite account.
2. **WXR file path** — the WordPress export XML.
3. **Old domain** — e.g. `https://oldsite.com`. Used to identify internal links to rewrite.
4. **Author handling** — single byline for all posts, or map WP authors to a field?
5. **Image handling** — do they want images re-hosted on BrightSite media (slower, fully migrated) or left pointing at the WP URLs (faster, but breaks when WP is taken down)? **Recommend re-hosting.**
6. **Draft confirmation** — this skill imports as `draft` only so the user can spot-check before going live.

## Workflow

### Step 1: Parse the WXR

WXR is RSS-flavored XML. Each `<item>` with `<wp:post_type>post</wp:post_type>` is a blog post. Extract:

- `<title>`
- `<link>` — the original URL (for redirect mapping)
- `<wp:post_name>` — the slug
- `<content:encoded>` — the body (HTML, may contain shortcodes or Gutenberg blocks)
- `<excerpt:encoded>` — the excerpt
- `<wp:post_date_gmt>` — published date
- `<wp:status>` — `publish`, `draft`, `private`, etc.
- `<wp:postmeta>` with `_thumbnail_id` — feature image attachment ID, which you cross-reference against `<item>` entries with `<wp:post_type>attachment</wp:post_type>` to get the image URL

Skip items where `<wp:status>` is `trash`, `auto-draft`, or `inherit`.

### Step 2: Clean each post body

WordPress content includes a lot of junk. Before creating the BrightSite post, clean:

- **Gutenberg block comments** like `<!-- wp:paragraph -->` and `<!-- /wp:paragraph -->` — strip them, keep the inner HTML.
- **Shortcodes** like `[caption ...]...[/caption]`, `[gallery]`, `[embed]`. For `[caption]`, convert to a `<figure><figcaption>` pair. For `[embed]` URLs to YouTube/Vimeo, replace with an `<iframe>`. For unknown shortcodes, leave a `<!-- TODO: shortcode [name] -->` comment so the user can fix manually.
- **Internal links** pointing at `https://oldsite.com/...` — rewrite to relative URLs.
- **Image URLs** — if re-hosting (recommended), upload each image to BrightSite media via `mcp__brightsite__request_upload` → HTTP PUT the raw bytes to the returned `upload_url` → `mcp__brightsite__complete_upload`, then rewrite the `src` to the new BrightSite media URL. `mcp__brightsite__complete_upload` requires `account_id`, `file_id`, `file_name`, and `name`; include `content_type`, `size`, `width`, `height`, and `folder_id` when available. The completed upload returns a media file `id` — use that as `feature_image_id` when creating the post (preferred over `feature_image_url`, which is for external URLs only).

### Step 3: Create one sample post first

Call `mcp__brightsite__create_post` for the first post (always as `draft` initially), passing:

- `account_id`
- `title`
- `slug` — use `wp:post_name` exactly. If it conflicts with an existing post slug, append `-2`.
- `content` — cleaned HTML
- `excerpt` — from `excerpt:encoded`, or auto-generated from first 160 chars if empty
- `published_at` — from `wp:post_date_gmt` (ISO 8601)
- `feature_image_url` or `feature_image_id` — depending on whether you re-hosted
- `meta_title` — typically the post title; if the WP post used Yoast/RankMath, those fields are in `<wp:postmeta>` under `_yoast_wpseo_title` or `rank_math_title`
- `meta_description` — similarly from `_yoast_wpseo_metadesc` or `rank_math_description`
- `status` — `"draft"`

`mcp__brightsite__create_post` does not return a fully-qualified URL, but the post path is `<blog_settings.blog_url>/<slug>` (e.g. `/blog/my-post`). Get the prefix from `mcp__brightsite__get_blog_settings`. Show the path to the user and ask them to confirm the formatting looks right before continuing.

### Step 4: Generate redirects

For each post where the new slug or URL path differs from the original, create a 301 redirect via `mcp__brightsite__create_redirect`:

- `source_path` — the relative path from the old URL, e.g. `/old-wp-path/old-slug`
- `target_url` — the new path, e.g. `/blog/new-slug`
- `redirect_type` — `301`
- `is_active` — `true`

Check `mcp__brightsite__get_blog_settings` for the `blog_url` and `listing_url` to know the target prefix.

### Step 5: Bulk import the rest

After the sample is approved, iterate the remaining posts. Throttle to ~3 posts/sec. Report progress every 25 posts.

### Step 6: Report

- Total draft posts imported
- Total images re-hosted
- Total redirects created
- A list of posts that had unresolved shortcodes (so the user can hand-fix)
- Next-step suggestion: spot-check 5 random posts in the editor, then run `mcp__brightsite__publish_post` on each (or use the bulk publish UI).

## Anti-patterns to avoid

- **Don't auto-publish.** Always import as draft, even if the user asks to publish during the import. Formatting issues sneak through and are easier to fix before anything is live. Note that `mcp__brightsite__create_post` writes content into staged fields regardless of `status` — going live requires a separate `mcp__brightsite__publish_post` call. So even `status: "published"` won't render to visitors until publish is called. Don't rely on that as a safety net, though: keep imports as `draft` so the user explicitly publishes after review.
- **Don't skip the redirect step.** The whole point of preserving SEO is that old URLs continue to resolve. A migration without redirects loses rankings.
- **Don't assume the WP slug is unique within BrightSite.** Check with `mcp__brightsite__list_posts` first.
- **Don't try to migrate WP comments.** BrightSite has its own comment system. If the client cares about comments, export them separately and discuss.

## Example invocation

> I have a WXR export at `~/clients/lawfirm/wordpress-export.xml`, 340 posts. Old domain is `https://oldlawfirm.com`. Account ID `YOUR_ACCOUNT_ID`. Re-host the images. Import as drafts.

## Tools used

- `mcp__brightsite__list_posts` — slug conflict check
- `mcp__brightsite__create_post` — create each post
- `mcp__brightsite__request_upload` / `mcp__brightsite__complete_upload` — re-host images
- `mcp__brightsite__create_redirect` — preserve old URLs
- `mcp__brightsite__get_blog_settings` — find the new blog URL prefix
