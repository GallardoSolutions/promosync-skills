# Changelog

All notable changes to the PromoSync skill. The version is the `version` field in
`.claude-plugin/plugin.json`; Claude Code only installs an update when that number changes, so every
release bumps it.

## 1.1.0 (2026-10-29)

### Fixed
- Tier prices in the cart now follow the store's Volume Pricing **Mode** (Per variant or Combined by
  product) through a theme setting and a shared `tier_qty` computation, so the cart shows the same price
  checkout charges.
- Removed the advice to enforce decoration run-charge tiers with a cart transform (Shopify Plus only).
  PromoSync's Quantity Price Breaks Function already discounts run-charge lines on any plan when Volume
  Pricing is on.
- Extra-color and extra-location run charges are now one cart line per extra at the product quantity, and
  every charge line carries `_promosync_parent` and `_promosync_charge` so the cart can keep charges in step
  with the product line (new pattern "Keep charge lines in step with the product line").
- Prices render with Shopify's `money` filters instead of a hardcoded dollar sign, with multi-currency
  guidance.
- Documented the `setup_tier` charge kind and its `psrestful.setup_tier_array` metafield.
- The Blank and Sample checkout limitation is stated plainly wherever those modes are described.
- The color swatch snippet used a `push` filter that does not exist in Shopify Liquid; it now matches labels
  with a nested loop. The gallery fallback images carry width and height. Storefront queries use
  `product(handle:)` instead of the deprecated `productByHandle`.

### Changed
- Removed internal implementation details. PromoSync settings are named as merchants see them, supplier
  notes are neutral, and no app internals are referenced.
- SKILL.md keeps six core facts; specialized topics are pointers into the reference files.
- Design guidance is tool-neutral (Figma and Shopify's own tools first).
- Added `version` to the plugin manifest and this changelog.

## 1.0.0 (2026-10-28)

- First public release: metafield catalog, theme app blocks, integration patterns, decoration pricing,
  design coordination, theme workflow (theme choice, Shopify CLI, GitHub, rate limits), notification emails,
  MSRP and shipping weight.
