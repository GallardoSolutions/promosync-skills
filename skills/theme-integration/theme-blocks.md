# PromoSync Theme App Blocks

PromoSync ships theme app blocks that merchants add through **Online Store > Themes > Customize** without
editing Liquid. This doc covers each block, what it reads, its settings, and where to place it, plus the
Quantity Price Breaks Shopify Function that the blocks rely on.

---

## 1. PromoSync Pricing Engine (embed): `promo-pricing-engine`

**Type:** Theme app embed block (not visible, runs theme-wide JS)

**What it does:** Foundation block. Wires up the client-side handlers for minimum quantity and tier pricing
that listen to variant-change events and quantity input changes. Other PromoSync blocks depend on it being
present.

**Metafields read:**
- `product.metafields.psrestful.minimum_quantity`: enforces HTML5 `min` on the qty input; prevents submit below minimum
- `variant.metafields.psrestful.part_price_array`: provides tier data to the blocks

**Settings:** Typically none beyond enable/disable.

**Positioning:** Enable as a theme app **embed** in the theme editor (not a block on a specific section). It runs site-wide.

**When to use:** Always enable if using any other PromoSync behavior; it's the substrate.

---

## 2. Tier Pricing Block: `tier-pricing`

**What it displays:** Volume pricing table (quantity to per-unit price).

**Metafields read:**
- `variant.metafields.psrestful.part_price_array`

**Behavior:** re-renders on variant change.

**Settings:**
- Display mode: collapsible / horizontal table / compact
- Heading text
- Show "save X%" column (optional)

**Positioning:** Product page, near or under the price block, before the quantity input.

**When to use:** Product pages for any PromoSync product with non-trivial volume breaks. Skip if all variants have flat pricing.

---

## 3. Location Decorations Block: `location-decorations`

> **Display and tagging only.** This block renders location and method **names** and writes
> `properties[Location]` / `properties[Decoration]`. It does **not** read the enriched `*VariantId` fields,
> compute setup, run, extra-color, extra-location or PMS pricing, or add decoration cart lines. **Decoration
> pricing requires custom theme code today**; see `decoration-pricing.md` and the "Not (yet) shipped" note
> below.

**What it displays:** Lets the shopper pick **where** on the product to add a decoration (for example "Front Chest") and **which method** (for example "Screen Print"). Selections flow into the cart as line item properties.

**Metafields read:**
- `product.metafields.psrestful.location_decorations`

**Settings:**
- **Display mode:**
  - `Dropdown selector`: cascading dropdowns (Location, then Decoration). Recommended for product pages with variant-like UX.
  - `Collapsible sections`: accordion per location, each showing available decorations as read-only info.
- Location label / placeholder
- Decoration label / placeholder
- Whether selection is required

**Output on add-to-cart (line item properties):**
- `properties[Location]`: selected `locationName`
- `properties[Decoration]`: selected `decorationName`
- Optionally `properties[Max Imprint Colors]`, `properties[Price Includes]`

**Positioning:** Product page, **after** the variant picker, **before** the quantity input and buy buttons.

**Install steps:**
1. Online Store > Themes > Customize
2. Navigate to the **Product page** template
3. Click **Add block** and search **PromoSync Decorations**
4. Set **Display mode** to `Dropdown selector`
5. Position after the variant picker, before quantity and buy buttons

**When to use:** Any decorated product. Skip for blank or undecorated SKUs.

---

## 4. Minimum Quantity Notice: `min-quantity-notice`

**What it displays:** Small informational banner: "Minimum order: N units."

**Metafields read:**
- `product.metafields.psrestful.minimum_quantity`

**Settings:** Typically only text customization (for example "Minimum order of {n} required").

**Positioning:** Product page, near the quantity input or above the buy buttons.

**When to use:** Products with `minimum_quantity > 1`. The **embed block** also enforces the minimum on the input; this block is the user-facing notice.

**Note:** Enforcement happens in the Pricing Engine embed. This block is just the visual notice. Both should typically be enabled together.

---

## 5. Volume Pricing Note: `volume-pricing-note`

**What it displays:** Informational call-out about volume pricing availability ("Save more when you buy in bulk").

**Metafields read:**
- `variant.metafields.psrestful.part_price_array` (to detect whether tiers exist)

**Settings:** Heading, body text.

**Positioning:** Product page, above the tier pricing table or in a sidebar.

**When to use:** When you want a visual teaser pointing to the tier table. Often paired with the Tier Pricing block. Skip if the tier table already has its own heading.

---

## 6. Star Rating: `star_rating`

**What it displays:** Star review rating (integrates with Shopify reviews or third-party apps).

**Metafields read:** None from PromoSync directly; uses standard `product.metafields.reviews.*` or similar.

**When to use:** Optional; not PromoSync-specific. Mentioned here for completeness because it ships with the app.

---

## Shopify Function: Quantity Price Breaks (`quantity-price-breaks`)

**Not a block**, but a companion extension that the blocks rely on.

**Type:** `purchase.product-discount.run` Shopify Function, distributed inside the PromoSync app, so it works on every Shopify plan (no Shopify Plus needed).

**What it does:** At checkout (and in the cart once Shopify applies automatic discounts), it iterates cart lines, reads `variant.metafield(namespace: "psrestful", key: "part_price_array")`, finds the best matching tier for the quantity, and applies a per-unit discount equal to the line's current price minus the tier price. It only ever lowers a price: a line whose price is already at or below the tier price is left alone. Decoration run-charge variants carry the same metafield, so their lines are discounted too.

**Mode** (PromoSync > Shop Settings > Volume Pricing > Mode) decides the quantity used for the tier lookup:

| Mode | Tier quantity |
|---|---|
| **Per variant** (default) | The line's own quantity |
| **Combined by product** | The sum of every cart line that shares the same parent product (all sizes and colors) |

**Requires:**
- **Enable Volume Pricing** on in PromoSync. The first save creates an automatic discount named **Quantity Price Breaks** in Shopify admin > Discounts; turning the setting off removes it. Don't edit that discount by hand.
- A populated `psrestful.part_price_array` on the variant (supplier tier data, or a Tier Pricing Rule).

**Implication for theme work:** The theme **does not** apply the discount itself; it only shows the discounted price so the customer sees their savings before checkout. The adjusted cart total in `main-cart-footer.liquid` is visual only, and it must use the same quantity rule as the Mode setting or the two will disagree. See `integration-patterns.md`, "Match the store's volume pricing mode".

---

## Not (yet) shipped as a block: Blank / Decorated / Sample selector

The blank/sample purchase mode trio (`psrestful.blank_minimum_quantity`, `psrestful.blank_part_price_array`, `psrestful.sample_price`) is a **gated** PromoSync feature: present only on Decorated-default shops where PSRESTful support has enabled Blank / Sample sales. It is **not** wrapped in a theme app block today; themes consume the metafields directly to render a Decorated / Blank / Sample selector on the product page.

When the gate is open, the storefront can let customers buy:
- **Decorated** (default, always available), using `minimum_quantity` plus `part_price_array`
- **Blank** (gated, may be absent on a specific product), using `blank_minimum_quantity` plus `blank_part_price_array`
- **Sample** (gated AND requires a qty-1 blank tier; may be absent even when Blank is available), using `sample_price` directly, qty forced to 1

Each mode's selection should write `properties[Configuration]=Decorated|Blank|Sample` to the cart line so the merchant can see and process the right configuration. The decorations block should be hidden in Blank/Sample mode.

**Checkout limitation, say it plainly:** the Quantity Price Breaks Function reads only `part_price_array`, so **Blank and Sample lines are charged at the variant's Shopify price at checkout**, not at the blank tier or sample price the theme displays. Until PromoSync enforces those prices, a store that takes payment at checkout should either keep Blank and Sample as quote or request flows, or price the difference at order processing. Do not ship a storefront that shows one price and charges another without the merchant knowing.

See `integration-patterns.md`, "Blank / Decorated / Sample purchase flows", for the full Liquid and JS pattern.

---

## Not (yet) shipped as a block: variant gallery

The per-variant media gallery (`variant.metafields.psrestful.gallery`) is a **gated** PromoSync feature (plans that include it, and only with the One Product (All Variants) grouping, not Split by Color). It is **not** wrapped in a theme app block today; themes consume the metafield directly.

If the shop qualifies, the theme should render a per-variant gallery from `psrestful.gallery` (featured image plus color-matched extras, in order) and fall back to `product.media` plus `variant.featured_media` when the metafield is empty.

See `integration-patterns.md`, "Variant gallery (gated)", for the Liquid and JS pattern.

---

## Not (yet) shipped as a block: decoration pricing / configurator

PromoSync's decoration-pricing surface (canonical decoration products for Setup Fee, Setup Tier, Run Charge, Extra Color Setup/Run, Extra Location Setup/Run and PMS Match, plus the enrichment job that stamps `*VariantId` GIDs onto each decoration entry in `psrestful.location_decorations`) is complete on the app side. The **storefront side is not wrapped in a theme app block**. The shipped `location-decorations` block (section 3) ignores the `*VariantId` fields entirely.

So to actually **price** a decoration on the storefront (compute setup, run tier, extra color, extra location and PMS, and add the corresponding cart lines), the theme needs **custom code**:

1. Read the enriched `location_decorations` and resolve each `*VariantId` (falling back to the canonical product's `psrestful.default_variant_id` when a per-product GID is absent).
2. Compute the per-kind pricing (run charges use `psrestful.part_price_array` tier ladders, setup tiers use `psrestful.setup_tier_array`, both in cents).
3. Add one cart line per applicable charge at the right quantity, each carrying `_promosync_parent` so it can follow the product line.
4. Make sure **Enable Volume Pricing** is on, so the Quantity Price Breaks Function discounts run-charge lines to the right tier at checkout (otherwise Shopify charges the headline qty-1 price).

See **`decoration-pricing.md`** for the full model and `integration-patterns.md`, "Decoration pricing (enriched `location_decorations`)", for adapt-ready JS.

---

## Recommended product-page stack

For a typical decorated promo product, enable (in order, top to bottom on the rendered page):

1. **PromoSync Pricing Engine** (embed; enables the substrate)
2. Variant picker (Shopify default)
3. **PromoSync Decorations** (`Dropdown selector` mode)
4. **Minimum Quantity Notice**
5. Quantity input (Shopify default; the embed enforces the minimum)
6. **Tier Pricing Block** (or **Volume Pricing Note** plus tier table)
7. Buy buttons

---

## Troubleshooting checklist

| Symptom | Check |
|---------|-------|
| Tier table empty | `variant.metafields.psrestful.part_price_array` has values; storefront access enabled on the metafield definition |
| Decoration dropdown missing options | `product.metafields.psrestful.location_decorations` populated; block `Display mode` is `Dropdown selector` |
| Min qty not enforced | Pricing Engine embed enabled; `minimum_quantity > 1` on the product; no other JS overriding the input's `min` |
| Discount not applied at checkout | **Enable Volume Pricing** is on in PromoSync; the **Quantity Price Breaks** automatic discount exists and is active in Shopify admin > Discounts; the variant has `part_price_array`; the Shopify price is above the tier price (the Function never raises prices) |
| Cart shows one tier price, checkout another | The theme's tier quantity does not follow the store's **Mode** (Per variant vs Combined by product). See `integration-patterns.md`, "Match the store's volume pricing mode" |
| Variant gallery empty on a qualifying shop | (a) Shop uses Split by Color grouping (gallery disabled by design); (b) storefront access not enabled on the `psrestful.gallery` metafield definition; (c) the supplier tagged no media against this part or its color (part-id-only suppliers never fall back to color). The theme should always fall back gracefully. See `metafields.md`, "How PromoSync matches supplier media to variants" |
| Blank/Sample selector not appearing | (a) Shop is Blank-default (Default Configuration Type = Blank): by design, the regular `minimum_quantity` and `part_price_array` already are the blank values; (b) Blank / Sample sales not enabled for the shop (ask PSRESTful support); (c) the supplier publishes no blank tier data for this product, so `blank_part_price_array` is nil on this variant. The theme must check the metafield itself |
| Sample option present but priced wrong, or missing when Blank is present | `sample_price` is **only** populated when the lowest tier of `blank_part_price_array` has `quantityMin == 1`. If the supplier's blank tiers start at qty 6, Sample is correctly omitted; do not fabricate a sample price from a higher tier |
| Blank/Sample line not discounted at checkout | Expected today: the Function only reads `psrestful.part_price_array`. Blank/Sample lines display the right price in the theme but are charged at the variant's Shopify price at checkout. Tell the merchant before shipping the selector |
| Decoration shown but no price / no `*VariantId` fields | The product is **unenriched**. Most common cause: missing `psrestful.extra_id` on the physical product. Backfill `extra_id` and run Location Decorations Enrichment again. See `decoration-pricing.md`, section 7 |
| A specific charge (for example extra-color) never appears | An **absent `*VariantId` means the charge doesn't apply** for this supplier and method; not a bug. First fall back to the canonical product's `psrestful.default_variant_id`; if that's also absent, skip the charge. Don't fabricate a price |
| Decoration run-charge line charged at the qty-1 price at checkout | **Enable Volume Pricing** is off, or the run-charge line's quantity is not the product quantity. With Volume Pricing on, the Function discounts run-charge lines like any other. See `decoration-pricing.md`, section 6 |
| Multi-color setup charged at the one-color price | Expected: `setup_tier_array` is display-only at checkout (a discount cannot raise a price). Handle multi-color setups with a draft order or at order processing |
| Tier prices showing the wrong magnitude ($0.25 instead of $25) | Use the `money` filter directly on `tier.price` (correct); do not divide by 100 first (wrong) |
| Prices wrong on a multi-currency store | Tier cents are in the store's base currency; Shopify Markets does not convert metafield values. See `integration-patterns.md`, "Currency" |
| Cart total doesn't match the product page tier | `main-cart-footer.liquid` not running the adjusted-total calculation; cart requires a refresh after an AJAX qty change |
