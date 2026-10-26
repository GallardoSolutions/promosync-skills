# PromoSync Theme App Blocks

PromoSync ships theme app blocks in the `promo-storefront-widgets` extension. Merchants add them through **Online Store → Themes → Customize** without editing Liquid. This doc covers each block, what it reads, settings, and best positioning.

---

## 1. PromoSync Pricing Engine (embed) — `promo-pricing-engine`

**Type:** Theme app embed block (not visible, runs theme-wide JS)

**What it does:** Foundation block. Wires up client-side logic (`MinQuantityHandler`, `TierPricingHandler`) that listens to variant-change events and quantity input changes. Other PromoSync blocks may depend on it being present.

**Metafields read:**
- `product.metafields.psrestful.minimum_quantity` — enforces HTML5 `min` on qty input; prevents submit below minimum
- `variant.metafields.psrestful.part_price_array` — provides tier data to JS

**Settings:** Typically none beyond enable/disable.

**Positioning:** Enable as a theme app **embed** in the theme editor (not a block on a specific section). It runs site-wide.

**When to use:** Always enable if using any other PromoSync behavior — it's the substrate.

---

## 2. Tier Pricing Block — `tier-pricing`

**What it displays:** Volume pricing table (quantity → per-unit price).

**Metafields read:**
- `variant.metafields.psrestful.part_price_array`

**Snippet dependencies:** `tier-pricing-data.liquid`, `tier-pricing-table.liquid`, `tier-pricing-horizontal.liquid`

**JS:** `tier-pricing.js` — re-renders on variant change.

**Settings:**
- Display mode: collapsible / horizontal table / compact
- Heading text
- Show "save X%" column (optional)

**Positioning:** Product page, near or under the price block, before quantity input.

**When to use:** Product pages for any PromoSync product with non-trivial volume breaks. Skip if all variants have flat pricing.

---

## 3. Location Decorations Block — `location-decorations`

> **v2-era / display + tagging only.** This shipped block renders location/method **names** and writes
> `properties[Location]` / `properties[Decoration]`. It does **not** read the enriched `*VariantId` fields,
> compute setup/run/extra-color/extra-location/PMS pricing, or add decoration cart lines. **Decoration
> *pricing* requires custom theme code today** — see `decoration-pricing.md` and the "Not (yet) shipped"
> note below.

**What it displays:** Lets the shopper pick **where** on the product to add a decoration (e.g., "Front Chest") and **which method** (e.g., "Screen Print"). Selections flow into cart as line item properties.

**Metafields read:**
- `product.metafields.psrestful.location_decorations`

**Snippet dependencies:** `location-decorations.liquid`

**Settings:**
- **Display mode:**
  - `Dropdown selector` — cascading dropdowns (Location → Decoration). Recommended for product pages with variant-like UX.
  - `Collapsible sections` — accordion per location, each showing available decorations as read-only info.
- Location label / placeholder
- Decoration label / placeholder
- Whether selection is required

**Output on add-to-cart (line item properties):**
- `properties[Location]` — selected `locationName`
- `properties[Decoration]` — selected `decorationName`
- Optionally `properties[Max Imprint Colors]`, `properties[Price Includes]`

**Positioning:** Product page, **after** the variant picker, **before** quantity input and buy buttons.

**Install steps:**
1. Online Store → Themes → Customize
2. Navigate to **Product page** template
3. Click **Add block** → search **PromoSync Decorations**
4. Set **Display mode** to `Dropdown selector`
5. Position after variant picker, before quantity/buy buttons

**When to use:** Any decorated product. Skip for blank/undecorated SKUs.

---

## 4. Minimum Quantity Notice — `min-quantity-notice`

**What it displays:** Small informational banner: "Minimum order: N units."

**Metafields read:**
- `product.metafields.psrestful.minimum_quantity`

**Settings:** Typically only text customization (e.g., "Minimum order of {n} required").

**Positioning:** Product page, near quantity input OR above the buy buttons.

**When to use:** Products with `minimum_quantity > 1`. The **embed block** also enforces the minimum on the input — this block is the user-facing notice.

**Note:** Enforcement happens in `MinQuantityHandler` (part of the embed block). This block is just the visual notice. Both should typically be enabled together.

---

## 5. Volume Pricing Note — `volume-pricing-note`

**What it displays:** Informational call-out about volume pricing availability ("Save more when you buy in bulk").

**Metafields read:**
- `variant.metafields.psrestful.part_price_array` (to detect if tiers exist)

**Settings:** Heading, body text.

**Positioning:** Product page, above tier pricing table OR in a sidebar.

**When to use:** When you want a visual teaser pointing to the tier table. Often paired with the Tier Pricing block. Skip if tier table already has its own heading.

---

## 6. Star Rating — `star_rating`

**What it displays:** Star review rating (integrates with Shopify reviews / third-party apps).

**Metafields read:** None from PromoSync directly — uses standard `product.metafields.reviews.*` or similar.

**When to use:** Optional; not PromoSync-specific. Mentioned here for completeness because it ships in the same extension.

---

## Shopify Function: `quantity-price-breaks`

**Not a block** — but a companion extension that the blocks rely on.

**Type:** `purchase.product-discount.run` Shopify Function

**What it does:** At checkout, iterates cart lines, reads `variant.metafield(namespace: "psrestful", key: "part_price_array")`, finds the best matching tier for each line's quantity, and applies a per-unit discount equal to `variant.price - tier.price`. Uses `discountApplicationStrategy: "ALL"`.

**Requires:**
- App scope: `write_discounts`
- Shop feature flag: `enable_volume_pricing = true`

**Implication for theme work:** The theme **does not** apply the discount itself; it only shows the discounted price so the customer sees their savings before checkout. The adjusted cart total in `main-cart-footer.liquid` is visual only. The real discount appears at checkout.

---

## Not (yet) shipped as a block: Blank / Decorated / Sample selector

The blank/sample purchase mode trio (`psrestful.blank_minimum_quantity`, `psrestful.blank_part_price_array`, `psrestful.sample_price`) is a **gated** PromoSync feature — only present on Decorated-default shops with `enable_blank_sample_sales=true`. It is **not** wrapped in a theme app block today; themes consume the metafields directly to render a Decorated / Blank / Sample selector on the product page.

When the gate is open, the storefront should let customers buy:
- **Decorated** (default, always available) — using `minimum_quantity` + `part_price_array`
- **Blank** (gated, may be absent on a specific product) — using `blank_minimum_quantity` + `blank_part_price_array`
- **Sample** (gated AND requires qty-1 blank tier — may be absent even when Blank is available) — using `sample_price` directly, qty forced to 1

Each mode's selection should write `properties[Configuration]=Decorated|Blank|Sample` to the cart line so the merchant can see and process the right configuration. The decorations block should be hidden in Blank/Sample mode.

The Shopify Function (`quantity-price-breaks`) does **not** apply blank or sample prices at checkout — only `part_price_array`. Blank/Sample pricing is display-only from the theme's perspective; the merchant's order processing handles the actual fulfillment price.

See `integration-patterns.md` → "Blank / Decorated / Sample purchase flows" for the full Liquid + JS pattern.

---

## Not (yet) shipped as a block: variant gallery

The per-variant media gallery (`variant.metafields.psrestful.gallery`) is a **gated** PromoSync feature (paid plans only, not for `by_color` shops). It is **not** wrapped in a theme app block today — themes consume the metafield directly.

If the merchant is on a qualifying plan, the theme should render a per-variant gallery from `psrestful.gallery` (featured image + color-matched extras, in order) and fall back to `product.media` + `variant.featured_media` when the metafield is empty.

See `integration-patterns.md` → "Variant gallery (gated)" for the Liquid + JS pattern.

---

## Not (yet) shipped as a block: decoration pricing / configurator

PromoSync's v3 decoration-pricing surface — canonical decoration products (Setup Fee, Run Charge, Extra
Color Setup/Run, Extra Location Setup/Run, PMS Match) plus the enricher that stamps `*VariantId` GIDs onto
each decoration entry in `psrestful.location_decorations` — is **fully implemented in the PromoSync
backend**. But the **storefront side is not wrapped in a theme app block**. The shipped
`location-decorations` block (§3) is v2-era and ignores the `*VariantId` fields entirely.

So to actually **price** a decoration on the storefront (compute setup + run-tier + extra-color +
extra-location + PMS and add the corresponding cart lines), the theme needs **custom code**:

1. Read the enriched `location_decorations` and resolve each `*VariantId` (falling back to the canonical
   product's `psrestful.default_variant_id` when a per-product GID is absent).
2. Compute the per-kind pricing (run charges use `psrestful.part_price_array` tier ladders, in cents).
3. Add one cart line per applicable charge at the right quantity.
4. Enforce run-charge tier prices at checkout via a cart-transform Shopify Function or draft order
   (otherwise Shopify charges the headline qty-1 price).

See **`decoration-pricing.md`** for the full model and `integration-patterns.md` → "Decoration pricing
(v3 enriched `location_decorations`)" for adapt-ready JS.

---

## Recommended product-page stack

For a typical decorated promo product, enable (in order, top → bottom on the rendered page):

1. **PromoSync Pricing Engine** (embed — enables substrate)
2. Variant picker (Shopify default)
3. **PromoSync Decorations** (`Dropdown selector` mode)
4. **Minimum Quantity Notice**
5. Quantity input (Shopify default — handler enforces min)
6. **Tier Pricing Block** (or **Volume Pricing Note** + tier table)
7. Buy buttons

---

## Troubleshooting checklist

| Symptom | Check |
|---------|-------|
| Tier table empty | `variant.metafields.psrestful.part_price_array` has values; storefront access enabled on metafield definition |
| Decoration dropdown missing options | `product.metafields.psrestful.location_decorations` populated; block `Display mode` is `Dropdown selector` |
| Min qty not enforced | Pricing Engine embed enabled; `minimum_quantity > 1` on product; no other JS overriding the input's `min` |
| Discount not applied at checkout | Shop `enable_volume_pricing=true`; app has `write_discounts` scope; discount function deployed and active |
| Variant gallery empty on a known-paid shop | (a) Shop is on `by_color` grouping (gallery disabled by design); (b) storefront access not enabled on `psrestful.gallery` metafield definition; (c) supplier media association strategy returned nothing for this part — HIT (`PartIdOnlyMediaAssociationStrategy`) never falls back to color; SanMar/S&S/default cascade through partId → color → fallback color and may still come up empty if the supplier didn't tag the media. Theme should always fall back gracefully. See `metafields.md` → "Media association strategies". |
| Blank/Sample selector not appearing | (a) Shop is Blank-default (`default_configuration_type='Blank'`) — by design, the regular `minimum_quantity`/`part_price_array` already are the blank values; (b) `enable_blank_sample_sales=false` (admin-only flag); (c) supplier didn't ship blank tier data for this product — gate open but `blank_part_price_array` is nil on this variant. Theme must check the metafield itself, not just the shop flag. |
| Sample option present but priced wrong, or missing when Blank is present | `sample_price` is **only** populated when the lowest tier of `blank_part_price_array` has `quantityMin == 1`. If the supplier's blank tiers start at qty 6, Sample is correctly omitted — do not fabricate a sample price from a higher tier. |
| Blank/Sample line not discounted at checkout | Expected — the `quantity-price-breaks` Shopify Function only reads `psrestful.part_price_array` (decorated tiers). Blank/Sample lines display the right price in the theme but are charged at the variant's base price at checkout; the merchant must override pricing at order processing or build a separate Function. Out of scope for the theme. |
| Decoration shown but no price / no `*VariantId` fields | The product is **unenriched**. Most common cause: missing `psrestful.extra_id` on the physical product (the enricher needs it to fetch per-product pricingHashes — without it it writes no `*VariantId`). Backfill `extra_id` and re-run the enricher. See `decoration-pricing.md` §7. |
| A specific charge (e.g. extra-color) never appears | An **absent `*VariantId` means the charge doesn't apply** for this supplier/method — not a bug. First fall back to the canonical product's `psrestful.default_variant_id`; if that's also absent, skip the charge. Don't fabricate a price. |
| Decoration run-charge line charged at qty-1 price at checkout | Expected without enforcement — Shopify charges the variant's headline price. Enforce the `part_price_array` tier with a cart-transform Shopify Function or a draft order. The theme's displayed tier price is otherwise display-only. See `decoration-pricing.md` §6. |
| Tier prices showing wrong magnitude ($0.25 instead of $25) | Using `money_without_currency` filter directly on tier.price (correct), not dividing by 100 first (wrong) |
| Cart total doesn't match product page tier | `main-cart-footer.liquid` not running the adjusted-total calculation; cart requires refresh after AJAX qty change |
