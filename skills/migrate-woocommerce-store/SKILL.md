---
name: migrate-woocommerce-store
description: Migrate a WooCommerce store from WordPress to BrightSite — catalog, product images, page content and redirects — rebuilding the theme as HEEx so the site looks the same after the move. Use when the user mentions "migrate WooCommerce," "move my store off WordPress," "WooCommerce to BrightSite," "migrate an online shop," "my client's store is on WordPress," or has a WooCommerce site they want rehosted. For a blog-only WordPress site with no shop, use migrate-wordpress-blog instead.
---

# Migrate a WooCommerce store

Move a WooCommerce store to BrightSite: products, images, page content, and the
URLs that already rank. The goal is a site that looks the same to a visitor and
keeps working after WordPress is switched off.

## Before you start

Ask the user for:

1. **Account ID** — the destination BrightSite account.
2. **The live site URL** — the migration reads the public site, so it needs no
   WordPress credentials.
3. **WooCommerce API keys** *(optional)* — only needed for real SKUs, cost
   prices and stock counts. Without them, prices, stock status and descriptions
   still come through; SKUs and exact quantities do not.
4. **Whether the shop is the whole site**, or a shop bolted onto a content site.
   That decides how much page rebuilding is involved.

## Step 1: survey the site before promising anything

Run these first. They decide whether the job is hours or days.

```bash
curl -s "https://SITE/wp-json/wp/v2/pages?per_page=1"   # X-WP-Total header = page count
curl -s "https://SITE/wp-json/wp/v2/product?per_page=1" # product count
curl -s "https://SITE/wp-json/wp/v2/posts?per_page=1"   # blog post count
curl -s https://SITE | grep -oE 'wp-content/themes/[a-z0-9-]+|elementor|et_pb_|wpb_'
```

Report to the user before building:

- **Product count** and whether any are variable products (see Step 3).
- **Page builder?** If the grep finds `elementor`, `et_pb_` (Divi) or `wpb_`
  (WPBakery), say so plainly: builder output is thousands of generated
  divs and inline styles, and a faithful rebuild is a redesign, not a
  migration. Quote that honestly rather than discovering it halfway.
- **Plugin features with no BrightSite equivalent.** Donations (GiveWP),
  bookings, memberships, wishlists and product comparison do not migrate.
  List them as out of scope up front.

## Step 2: extract everything to a JSON bundle first

Extract to a file, then import from the file. Two passes, not one: it makes the
import repeatable and lets the user check what was found before anything is
written.

The public REST API gives you pages, posts and product names. It does **not**
give you prices — those come from the JSON-LD on each product page:

```
GET /wp-json/wp/v2/product?per_page=100   → id, slug, title, content
GET  https://SITE/product/<slug>/         → <script type="application/ld+json">
                                             carries price, availability, sku, image
```

Two traps that cost real time:

- **The REST content field is often empty.** Themes with custom page templates
  (`page-template-template-about`) render content from PHP, not `post_content`.
  If `content.rendered` is empty but the live page clearly has text, fetch the
  rendered HTML and parse the sections out of it. Always capture the rendered
  HTML alongside the REST fields.
- **`sku` in JSON-LD falls back to the WordPress post ID** when the product has
  no real SKU. If `sku` equals the numeric post id, treat it as absent rather
  than importing 4693 as a SKU.

Paginate media with `?page=N` until a page comes back empty. Do not stop when a
page returns fewer rows than you asked for — WordPress caps `per_page` well
below 100, so a short page is normal and is not the end.

## Step 3: create the catalog

Products need a store first. If `list_products` errors, the account has no
store or commerce is not enabled for it — tell the user; it is an admin action,
not something the skill can do.

For each product, one call:

```
create_product(account_id, title, slug, description, price_amount,
               inventory_quantity, featured_image_id, seo_description)
```

- **`price_amount` is minor units.** $15.00 is `1500`. Never `15.0`.
- `create_product` creates the default variant for you. A simple
  WooCommerce product is one call — no separate variant step.
- Import the WooCommerce **slug unchanged**. It is what search engines have
  indexed, and BrightSite serves products at `/products/{slug}`.
- Products are created as `draft`. Call `set_product_status(id, "active")` once
  the user has checked them. Activation fails if the variant has no price,
  which is the intended guard, not an error to work around.
- Map `availability`: `InStock` → a positive quantity, `OutOfStock` → `0`.

**Variable products** (a `data-product_variations` attribute on the product
page) carry sizes or colours. `create_product` gives you one default variant,
which is wrong for them. Create the product, then `add_product_option` and
`generate_variants`, and price each variant. Say plainly which products these
are rather than importing them as single-price and leaving the user to find out.

Recreate WooCommerce product categories as collections with `create_collection`,
then attach products to them.

## Step 4: rehost every image, including the ones inside descriptions

This is the step that decides whether the site survives WordPress being turned
off, and it is the one most often half-done.

A product's `featured_image` is obvious. The images **inside the description
HTML** are not, and there are usually more of them. Rehost both:

1. `request_upload` → `upload_url` and `file_id`
2. HTTP `PUT` the raw bytes to `upload_url` with the right `Content-Type`
3. `complete_upload` with `file_id`, `file_name`, `name`, and `size`
4. Rewrite the `src` in the description to the new BrightSite media URL

Then strip `srcset` and `sizes` from the rewritten `<img>` tags. Those still
list WordPress URLs, and a browser will happily pick one, so a page that looks
migrated still hotlinks the old host.

**Validate the bytes before creating the record.** A 404 from WordPress returns
an HTML error page with a 200-ish shape; upload it and you get a media record
that looks fine in a list and is broken on the page. Check for a real image
signature (`FF D8` JPEG, `89 50 4E 47` PNG, `RIFF….WEBP`) and fail loudly.

Guessing an image URL from a filename does not work — WordPress files live under
`/uploads/YYYY/MM/`, and the same filename can appear under several dates. Take
the URL from the page HTML, never construct it.

Finish by grepping every imported description for the old domain. The number
should be zero.

## Step 5: rebuild the pages

The theme does not come across. What comes across is content, structure and the
design tokens; the HEEx is written fresh.

Capture the tokens from the live site before writing anything:

- **Colours** — usually in an inline `<style>` block from the theme's
  customiser, not the stylesheet.
- **Fonts** — count `font-family` in the theme CSS; the most frequent stack is
  the body font, the serif one is usually headings.
- **Logo** — the `<img class="custom-logo">` tag.
- **Section structure** — each `<section>` on the rendered page is one
  section in the rebuild.

Then author the pages per the `visual-editor-authoring` skill: `data-bs-edit`
on every piece of text the client will want to change, and `params_schema`
collections for anything repeating (team members, feature cards, testimonials).
A migration that produces a pixel-perfect page the client cannot edit has moved
the problem, not solved it.

Skip the WooCommerce plumbing pages entirely. Cart, checkout, my-account,
order-received, wishlist and compare are all handled by BrightSite's storefront
templates. Only the real content pages need rebuilding, and on most shops that
is three or four.

## Step 6: redirects

Product URLs change shape. Create a 301 for each:

| WordPress | BrightSite |
|---|---|
| `/product/<slug>` | `/products/<slug>` |
| `/product-category/<slug>` | `/collections/<slug>` |
| `/shop` | `/collections/<main-collection>` |

Add a redirect for any page whose slug changed, and for removed plugin pages
(`/wishlist`, `/compare`) so an indexed URL lands somewhere sensible.

## Step 7: report

Give the user:

- Products imported, and which are still `draft`.
- Images rehosted, and **the count of remaining references to the old domain**
  (should be zero).
- Pages rebuilt, and which were skipped as storefront plumbing.
- Redirects created.
- **What did not migrate** — donations, bookings, memberships, reviews,
  customer accounts, past orders. Be explicit. These are the things that
  surface a week later.

## Anti-patterns to avoid

- **Do not trust `content.rendered`.** An empty body on a themed page means the
  content is in PHP, not that the page is empty.
- **Do not import a price as a decimal.** `price_amount` is cents.
- **Do not leave `srcset` behind** after rewriting an image `src`.
- **Do not construct image URLs** from filenames. Read them from the page.
- **Do not upload bytes without checking they are an image.**
- **Do not activate products silently.** Import as draft, let the user check
  prices and stock, then activate.
- **Do not promise visual fidelity on a page-builder site** before looking. On
  a stock theme it is achievable; on Elementor or Divi it is a redesign.
- **Do not migrate the WooCommerce plumbing pages.** BrightSite has its own.
- **Do not claim customer accounts or past orders came across.** They do not.
