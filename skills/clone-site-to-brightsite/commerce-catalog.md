# Commerce catalog import

Read this when Phase 1 fingerprinting found a shop on the source and the user
wants the catalog in scope. It covers getting products, variants and collections
into BrightSite. The storefront's *appearance* is not here — that is cloned like
any other section, from the main skill.

Nothing below is WordPress-specific except where it says so. Extracting a
catalog from Shopify, Squarespace or a CSV lands in the same MCP calls with the
same traps.

## Before anything: does the account have a store?

If `list_products` errors, commerce is not enabled for the account. That is an
admin action — tell the user, do not try to work around it. Everything else here
fails until it is on.

## Extract to a file first, then import from the file

Two passes, not one. It makes the import repeatable after a failure, and it lets
the user check what was found before a single record is written. A half-finished
loop against a live account is much worse than a re-run.

### WooCommerce

The public REST API needs no credentials and gives you names, slugs and bodies.
It does **not** give you prices — those come from the JSON-LD on each product
page:

```
GET /wp-json/wp/v2/product?per_page=100&page=N   → id, slug, title, content
GET /product/<slug>/                             → <script type="application/ld+json">
                                                    price, availability, sku, image
```

Paginate with `?page=N` until a page comes back **empty**. WordPress caps
`per_page` well below 100, so a short page is normal and is not the end of the
list — stopping early silently truncates the catalog.

Two traps that cost real time:

- **`content.rendered` is often empty.** Themes with custom page templates
  render from PHP, not `post_content`. An empty body on a page that visibly has
  text means the content is in the template, not that the product has no
  description. Capture the rendered HTML alongside the REST fields.
- **`sku` in JSON-LD falls back to the WordPress post ID** when the product has
  no real SKU. If `sku` is the numeric post id, treat it as absent rather than
  importing `4693` as a SKU.

WooCommerce API keys are optional and only buy you real SKUs, cost prices and
exact stock counts. Prices, availability and descriptions all come through
without them.

### Variable products

A `data-product_variations` attribute on the product page means sizes, colours
or similar. Note which products these are during extraction — they need a
different import path (below), and discovering them mid-import means going back.

## Creating products

One call per simple product:

```
create_product(account_id, title, slug, description, price_amount,
               sku, inventory_quantity, compare_at_amount,
               featured_image_id, seo_description)
```

`sku` and `compare_at_amount` seed the default variant on the same call — no
follow-up `update_variant` needed for a simple product. `compare_at_amount` is
where a WooCommerce "regular price" goes when the product is on sale, with
`price_amount` holding the sale price.

- **`price_amount` is minor units.** $15.00 is `1500`, never `15.0`. This is the
  single most expensive mistake in this file: a decimal silently imports a
  catalog priced at one hundredth of its value, and it looks plausible in a list.
- **`create_product` creates the default variant for you.** A simple product is
  one call; there is no separate variant step to remember.
- **Keep the source slug unchanged.** It is what is indexed, and BrightSite
  serves products at `/products/{slug}`. If you do change one later,
  `update_product` writes the 301 from the old product URL automatically — do
  not hand-create that redirect as well.
- **Products are created as `draft`.** Call `set_product_status(id, "active")`
  only after the user has checked prices and stock. Activation validates the
  product is sellable and fails when there is no active variant with a price —
  that is the intended guard, not an error to route around.
- Map availability: `InStock` → a positive quantity, `OutOfStock` → `0`.

### Variable products need three more calls

`create_product` gives one default variant, which is wrong for a product with
options. Create the product, then `add_product_option`, then `generate_variants`,
then price each variant with `update_variant`. Variant ids come from
`get_product` — `list_products` returns products only, without their variants.

Say plainly which products these are, rather than importing them as
single-price and leaving the user to discover it. An eight-size product imported
at one price is a pricing bug, not a cosmetic one.

### Collections need three calls or they are invisible

Recreate the source's product categories:

1. `create_collection`
2. `add_products_to_collection` — products already in the collection are
   skipped, so a re-run after a failure does not duplicate them
3. `set_collection_status(active)`

A collection is created as **draft and holding no products**. Miss either of the
last two and its storefront page is empty or unreachable — and it will look like
the import worked.

### Where products can and cannot be listed

`product-grid` reads a `products` assign that the renderer only fills on a
**collection page** (`/collections/<slug>`). Embedding
`component("product-grid")` on an ordinary CMS page renders "No products yet."
no matter how many active products exist.

So link the homepage to a collection, or feature products with hand-authored
markup. Do not promise a homepage product grid.

## Images

Rehost every product image, including the ones inside descriptions, through the
main skill's flow: `request_upload` → HTTP PUT the raw bytes → `complete_upload`,
reusing the same source-URL → media-id map so nothing uploads twice.

The rules from the main skill apply unchanged and matter more here, because a
catalog has hundreds of images:

- Take URLs from the rendered page. Never construct them from a filename —
  WordPress files live under `/uploads/YYYY/MM/` and the same filename appears
  under several dates.
- Check the bytes are an image (`FF D8` JPEG, `89 50 4E 47` PNG, `RIFF…WEBP`)
  before uploading. A 404 from the source CMS is an HTML page, and uploading it
  produces a media record that lists fine and is broken on the page.
- **Rewrite `srcset` as well as `src`.** A rewritten `src` with a stale `srcset`
  serves the old domain's image to most browsers, and looks correct in a
  desktop screenshot.

Finish by grepping every imported description for the old domain. The count
should be zero.

## Rate limits

The MCP endpoint allows **100 requests per minute**. Fifty images is roughly 150
calls across `request_upload`, the PUT and `complete_upload`, so a straight loop
hits the ceiling and starts failing with HTTP 429. Back off exponentially and
retry rather than dropping the asset — a dropped asset is a broken image nobody
notices until handover.

## Redirects

Product URLs change shape:

| WordPress | BrightSite |
|---|---|
| `/product/<slug>` | `/products/<slug>` |
| `/product-category/<slug>` | `/collections/<slug>` |
| `/shop` | `/collections/<main-collection>` |

Add a redirect for any page whose slug changed, and for removed plugin pages
(`/wishlist`, `/compare`) so indexed URLs land somewhere sensible.

## Skip the plumbing pages

Cart, checkout, my-account, order-received, wishlist and compare are all handled
by BrightSite's storefront templates. Do not clone them as pages. On most shops
that leaves three or four real content pages for the main skill to rebuild.

## What does not migrate — say so up front

Be explicit in the Phase 6 report. These are what surface a week later:

- Customer accounts and past orders
- Product reviews
- Donations (GiveWP), bookings, memberships
- Wishlists and product comparison
- Subscriptions and any recurring billing

Plugin features with no BrightSite equivalent are out of scope, and listing them
before the build is much cheaper than explaining them after it.

## Anti-patterns

- **Do not import a price as a decimal.** `price_amount` is minor units.
- **Do not trust an empty `content.rendered`.** The content is in PHP.
- **Do not treat a numeric `sku` matching the post id as a real SKU.**
- **Do not stop paginating on a short page.**
- **Do not import variable products as single-price.**
- **Do not create a collection and walk away** — it is draft and empty until two
  more calls land.
- **Do not activate products silently.** Draft, review, then activate.
- **Do not leave `srcset` pointing at the old domain.**
- **Do not upload bytes without checking they are an image.**
- **Do not claim customer accounts, orders or reviews came across.** They do not.
