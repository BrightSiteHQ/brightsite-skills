---
name: migrate-woocommerce-store
description: Migrate a WooCommerce store from WordPress to BrightSite — catalog, product images, page content and redirects — rebuilding the theme so the site looks the same after the move. Use when the user mentions "migrate WooCommerce," "move my store off WordPress," "WooCommerce to BrightSite," "migrate an online shop," "my client's store is on WordPress," or has a WooCommerce site they want rehosted. This skill routes into clone-site-to-brightsite, which owns the workflow.
---

# Migrate a WooCommerce store

**This workflow lives in the `clone-site-to-brightsite` skill.** Load that and
follow it — do not run a separate migration from here.

A store move is a site clone that happens to include a catalog. Splitting them
meant surveying the source twice, rehosting images twice, and two sets of rules
about which one owned the pages. The clone skill now fingerprints the source
during Phase 1 reconnaissance, finds the shop, and pulls in the catalog import
from `commerce-catalog.md` in its own directory.

## What to do

1. Load **clone-site-to-brightsite**.
2. Give it the live store URL and the destination account id.
3. Its Phase 1 fingerprint reports the product count, the post count, and
   whether the source runs a page builder. Confirm scope from there.
4. The catalog rules — prices in minor units, variable products, the
   three-call collection sequence, the `sku`-falls-back-to-post-ID trap, rate
   limits — are in `clone-site-to-brightsite/commerce-catalog.md`.

## What carried over

Nothing was lost in the move. The commerce specifics live in
`commerce-catalog.md`; the survey, image rehosting, page rebuilding and QA were
duplicates of what the clone skill already did more carefully, and now happen
once.

## Related

- **clone-site-to-brightsite** — the actual workflow, including the storefront
  design and the catalog import.
- **migrate-wordpress-blog** — a posts-only import from a WXR export file. Use
  it when the client wants their posts moved but not their design, or when
  WordPress is already switched off and there is no live site to measure.
