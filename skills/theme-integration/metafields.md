# PromoSync Metafield Catalog

Authoritative catalog of every Shopify metafield PromoSync writes, reads, or expects, and of the PromoSync settings that decide whether each one is written. Last refreshed: 2026-10-29. When something is missing here, check https://promosync-docs.psrestful.com.

Source of truth: the metafield definitions registered by the PromoSync app in the store.

---

## Namespaces

| Namespace | Owner types | When written |
|-----------|-------------|--------------|
| `psrestful` | Product, ProductVariant | Always, core sync |
| `mm-google-shopping` | Product, ProductVariant | Only when **Enable Google Merchant Metafields** is on in PromoSync settings |

---

## Product-level metafields

### `psrestful.*` (product)

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `supplier_code` | `single_line_text_field` | Supplier code | `"PCNA"`, `"SanMar"` |
| `product_id` | `single_line_text_field` | Supplier's product ID | `"B03042000"` |
| `extra_id` | `number_integer` | PromoSync's id for the product; links the Shopify product to its PSRESTful record | `12345` |
| `status` | `single_line_text_field` | Lifecycle status | `"active"`, `"draft"`, `"closeout"`, `"discontinued"` |
| `effective_date` | `date_time` | When product becomes available | `"2025-01-15T00:00:00"` |
| `export` | `boolean` | Export flag | `"true"` / `"false"` |
| `primary_material` | `single_line_text_field` | Main material | `"100% Polyester"` |
| `brand` | `single_line_text_field` | Brand | `"Nike"` |
| `country_of_origin` | `single_line_text_field` | ISO country | `"MX"`, `"USA"` |
| `is_caution` | `boolean` | Has caution warning | `"true"` / `"false"` |
| `caution_comment` | `multi_line_text_field` | Caution text | `"Contains latex."` |
| `line_name` | `single_line_text_field` | Product line | `"Premium Apparel"` |
| `minimum_quantity` | `number_integer` | Minimum order qty for the shop's default configuration (decorated when **Default Configuration Type** is Decorated, blank otherwise) | `5`, `25`, `100` |
| `blank_minimum_quantity` | `number_integer` | **Gated.** Minimum order qty for the blank (undecorated) configuration. Only on shops with Blank / Sample sales enabled; see "Blank / sample metafields" below | `1`, `6`, `12` |
| `lead_time` | `number_integer` | Lead time in days | `14` |
| `msrp_min` | `number_decimal` | **Gated, opt-in.** Lowest `psrestful.msrp` across the product's variants, for product cards and smart-collection rules. Same gating as `msrp`. See "MSRP" below | `"12.50"` |
| `is_on_demand` | `boolean` | On-demand product | `"true"` / `"false"` |
| `is_rush_service` | `boolean` | Rush service available | `"true"` / `"false"` |
| `imprint_size` | `single_line_text_field` | Imprint area size | `"1.5\" x 1.5\""` |
| `price_expires_date` | `date_time` | Price expiry | `"2025-12-31T23:59:59"` |
| `is_hazmat` | `boolean` | Hazmat flag | `"true"` / `"false"` / null |
| `default_set_up_charge` | `single_line_text_field` | Setup fee | `"$50"` or `"50"` |
| `default_run_charge` | `single_line_text_field` | Per-unit run charge | `"$0.25/unit"` |
| `unspsc_commodity_code` | `number_integer` | UNSPSC code | `23101500` |
| `location_decorations` | `json` | Decoration locations and methods | See shape below |
| `features` | `multi_line_text_field` | Feature bullets | `"Water resistant\nUV protected"` |
| `care_instructions` | `multi_line_text_field` | Care instructions | `"Machine wash warm"` |
| `charge_type` | `single_line_text_field` | Charge category | `"Setup"`, `"Run Charge"`, `"Order"` |
| `marketing_points` | `json` | Supplier `ProductMarketingPointArray` as-is: `[{pointType, pointCopy}]`. Not normalized: every supplier labels points differently and some include notes meant for distributors rather than shoppers. Written only when the supplier sends at least one point with copy. Never render unfiltered; see "Marketing points" below | `[{"pointType":"Bullet Point","pointCopy":"8 Panels"}]` |
| `color-swatches` | `list.metaobject_reference` | **Option-linked** list of `custom-color-swatch` metaobjects (each has `label` and `hex_color`) that power native Color-option swatches. See "Color swatches" below; the type is a list, not a scalar | `[gid://shopify/Metaobject/123, …]` |

### `location_decorations` JSON shape

This metafield exists in two states, both valid:

- **Unenriched (simple)**: what PromoSync writes on import. Just locations and method names. Use it for display-only decoration selectors.
- **Enriched**: after the **Location Decorations Enrichment** job runs, each decoration entry gains `*VariantId` fields pointing at canonical decoration-product variants, plus richer location fields (`locationId`, `decorationsIncluded`, `minDecoration`, `maxDecoration`, `locationRank`). This is what lets a theme price decorations (setup, run, extra color, extra location, PMS). It is the same metafield; enrichment adds fields in place.

Enriched example (one location, two decorations):

```json
[
  {
    "locationId": 39263,
    "locationName": "Front Chest",
    "decorationsIncluded": 1,
    "defaultLocation": true,
    "minDecoration": 0,
    "maxDecoration": 1,
    "locationRank": 1,
    "decorations": [
      {
        "decorationId": 58355,
        "decorationName": "Screen Print",
        "maxImprintColors": 6,
        "default": true,
        "priceIncludes": false,
        "setupFeeVariantId": "gid://shopify/ProductVariant/1001",
        "setupTierVariantId": "gid://shopify/ProductVariant/1008",
        "runChargeVariantId": "gid://shopify/ProductVariant/1002",
        "extraColorSetupVariantId": "gid://shopify/ProductVariant/1003",
        "extraColorRunVariantId": "gid://shopify/ProductVariant/1004",
        "extraLocationSetupVariantId": "gid://shopify/ProductVariant/1005",
        "extraLocationRunVariantId": "gid://shopify/ProductVariant/1006",
        "pmsMatchVariantId": "gid://shopify/ProductVariant/1007"
      },
      {
        "decorationId": 58350,
        "decorationName": "Embroidery",
        "maxImprintColors": 6,
        "default": false,
        "priceIncludes": false,
        "setupFeeVariantId": "gid://shopify/ProductVariant/2001",
        "runChargeVariantId": "gid://shopify/ProductVariant/2002"
      }
    ]
  }
]
```

Location-entry fields:
- `locationName`: user-facing label (for example "Front Chest", "Left Sleeve"). May have trailing whitespace; trim it.
- `locationId`: supplier's location id; don't surface it to customers.
- `decorationsIncluded`: decorations included before extra-location charges apply (feeds the extra-location math).
- `defaultLocation`: default-select this location in the UI when `true`.
- `minDecoration` / `maxDecoration`: min and max decorations selectable here. `maxDecoration: 0` means skip rendering.
- `locationRank`: display order (lower = earlier).
- `decorations[]`: array of methods available at that location.

Per-decoration fields:
- `decorationName`: **canonical** method name ("Screen Print", "Embroidery", "Laser Engraving", "Full Color", and so on), already alias-resolved by PromoSync.
- `decorationId`: supplier's decoration id; don't surface it.
- `maxImprintColors`: max distinct imprint colors supported (color-count UI ceiling).
- `default`: whether this decoration is pre-selected.
- `priceIncludes`: base product price already includes this decoration (skip setup and run charges when `true`).
- `setupFeeVariantId`, `setupTierVariantId`, `runChargeVariantId`, `extraColorSetupVariantId`, `extraColorRunVariantId`, `extraLocationSetupVariantId`, `extraLocationRunVariantId`, `pmsMatchVariantId`: **enrichment-only** GIDs of canonical decoration-product variants. An **absent** field means "this charge doesn't apply" (not "missing data"). camelCase exactly as written.

> Enrichment removes older fields on every run: the `*ProductId` pointers and the pre-split `extraColorVariantId` / `extraLocationVariantId` (replaced by the `*Setup` / `*Run` pairs). Don't read them.

For the full decoration-pricing model (canonical products, the eight charge kinds, variant resolution with the `default_variant_id` fallback, the setup, run, extra and PMS math, and multi-line cart-add) see **`decoration-pricing.md`**.

### `psrestful.*` (Liquid access)

```liquid
{{ product.metafields.psrestful.minimum_quantity }}
{{ product.metafields.psrestful.location_decorations.value }}  {% comment %} array of objects {% endcomment %}
{{ product.metafields.psrestful.supplier_code }}
```

### `marketing_points` (product): raw supplier copy, filter before display

Introduced September 2026.

**What it is.** The PromoStandards Product Data `ProductMarketingPointArray`, copied as JSON in its PromoStandards shape. Some suppliers put most of their spec sheet here (bullets, material, dimensions) instead of the part-level `specification_array`.

```json
[
  {"pointType": "Bullet Point", "pointCopy": "8 Panels, Automatic Open Mechanics, Steel Shaft"},
  {"pointType": "Additional Info", "pointCopy": "Price includes a single color, single panel decoration."},
  {"pointType": "Material", "pointCopy": "Nylon"},
  {"pointType": "Product Width (IN)", "pointCopy": "44"},
  {"pointType": "Eco-Friendly", "pointCopy": "0"}
]
```

**Write rules.**
- Product-level, type `json`, no setting or plan gate. Points whose `pointCopy` is empty are dropped and `pointCopy` is trimmed. If nothing is left, the metafield is **not written** (absent is normal, not a bug).
- `pointType` is passed through untouched. It can be `null` or free text.
- Written on **import** and by **Update Metafields**. Products imported before the metafield existed get it after the merchant runs Update Metafields on them.
- The definition is **not** storefront-exposed. Liquid can read it; Storefront API JS (`node(id:)`, `product(handle:)`) cannot.
- PromoSync does **no** filtering, mapping or normalization. Deciding what to show is theme work.

**How suppliers use it** (observed across a 40-supplier catalog in October 2026; feeds change, so re-check a few products per supplier before relying on a row):

| Supplier (code) | `pointType` values seen | Display |
|---|---|---|
| Peerless, Stromberg (`STROMBERGBRAND`), Diamondback (`DIAMONDBACKBRANDING`), WOWLine (`WOW`), SugarSpot, Raining Rose, Coloring Book Solutions, most suppliers with numeric codes | `Bullet Point` plus `Additional Info`, `Material`, `Product Width/Length/Height (IN)`, `Product Weight (LB)`, 0/1 flags (`Eco-Friendly`, `Woman Owned`, `PromoCares Collection`, `QCA Certified Supplier`), `Industry`, `Must Be Ordered In Increments Of` | Show `Bullet Point`; optionally measurements and material as label: value rows; hide the rest |
| Primeline (`PRIME`), Ariel, Gemline (`GEM`) | `Feature` (Gemline also `ECOCertification`, `CarbonFootprint`, `Product`) | Show `Feature` |
| HIT Promotional Products (`HIT`) | `Highlight`, on a minority of products | Show `Highlight` |
| Koozie Group (`BIC`) | `BTH`, a few per product, real product bullets ("Snap-on slider lid", "80 sheets", "Assembled in USA") | Show as bullets; skip the few that start with `Top markets:` (notes for distributors). Some mention optional add-on charges, which can conflict with the store's own option pricing |
| Logomark (`LMRK`) | `Feature`, `Quote`, `PROP 65` | Show `Feature` |
| Showdown Displays (`sign`) | `pointType` is `null`, on some products: always a single short marketing tagline | Show as a tagline or intro line, not a bullet |
| TerryTown (`terry`) | The bullet text arrives in `pointType` rather than `pointCopy` | Hide until the feed changes |
| Starline (`STAR`) | `Packaging`, `Production Time`, `Price`, `Material`, `NOTE`, `Output` and others | Mostly notes for distributors; hide `Price`, `NOTE`, `Production Time` |
| Snugz (`SNUGZ`) | `Compliance` only | Hide, or a dedicated compliance block |
| Goldstar | `HTS US`, `HTS CA`, `HTS EU` (customs codes) | Hide |
| PCNA | Dozens of catalog attributes per product (`PCNAAttributes-*`, `PCNAInternal-*`) | Hide everything; not product copy |
| NuPromo, Apli | `Additional Info` (plus `Shipping`) | Hide (distributor notes) |
| Many others | Send nothing | Metafield absent |

**Takeaways for theme work.**
1. **Allow-list, never deny-list.** Show only known customer-facing types; everything else stays hidden by default. A deny-list would leak a supplier's internal catalog attributes.
2. **Match several bullet names.** A `Bullet Point`-only filter renders nothing for HIT, Primeline, Ariel, Gemline and Koozie Group. A reasonable starting allow-list is `Bullet Point`, `Feature`, `Highlight`, `BTH`. Untyped (`null`) points are a single tagline, so render them separately as an intro line.
3. **0/1 flags read wrong on a product page.** `Eco-Friendly: 0` means *not* eco-friendly. Never print flag types as text.
4. **`Additional Info` is usually a distributor note** (pricing or decoration terms that may contradict the store's own decoration pricing). Hide it by default.
5. **Expect absence.** Many suppliers send nothing, others only for some products. Always `if points` / `points.size > 0` and render nothing (or fall back to `product.description`).
6. Supplier usage drifts. Re-check a couple of products per supplier when adding a supplier. PSRESTful's Data Explorer shows the raw `ProductMarketingPointArray` per product.

Pattern: `integration-patterns.md`, "Marketing points (product specs)".

### Canonical decoration products (`psrestful.*`)

PromoSync's **Insert Decorations** action creates a shop-wide catalog of canonical decoration products, one Shopify product per (imprint method × charge kind), vendor `PromoSync`, tagged `[<charge_kind>, promosync_decoration]`. They hold the actual decoration prices; physical products point at their variants via the `*VariantId` fields on `location_decorations` (above). These are separate Shopify products, not metafields on the physical product.

**Product-level metafields** (namespace `psrestful`):

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `method_slug` | `single_line_text_field` | Canonical imprint-method slug (lookup and grouping key) | `"screen-print"` |
| `charge_kind` | `single_line_text_field` | One of `setup_fee`, `setup_tier`, `run_charge`, `extra_color_setup`, `extra_color_run`, `extra_location_setup`, `extra_location_run`, `pms_match` | `"run_charge"` |
| `default_variant_id` | `variant_reference` | **Storefront fallback** variant for products with no per-product `*VariantId` (common for blank-goods suppliers). Auto-picked as the pricing class shared by the most products; the merchant can pin a different one in PromoSync | `gid://shopify/ProductVariant/667…` |

**Variant-level metafields** (one variant per pricing equivalence class; namespace `psrestful`):

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `pricing_hash` | `single_line_text_field` | 8-char hash identifying this pricing class across re-runs; also the SKU suffix | `"a1b2c3d4"` |
| `suppliers` | `json` | Supplier codes sharing this class (PromoSync's matching key; the theme rarely reads it) | `["ARIEL","HIT"]` |
| `part_price_array` | `json` | **Run-charge variants only.** Quantity-tier ladder, `price` in integer cents, ascending, equal-price tiers collapsed | `[{"quantityMin":1,"price":85},{"quantityMin":144,"price":70}]` |
| `setup_tier_array` | `json` | **Setup-tier variants only.** Setup priced by decoration units: `uom` names the axis (`Colors` or `Stitches`), `tiers` is ascending by `minUnits`, `price` in integer cents, equal-price tiers collapsed | `{"uom":"Colors","tiers":[{"minUnits":1,"price":5500},{"minUnits":2,"price":9500}]}` |

Handle = `slugify(decorationName) + suffix` where suffix is `-setup-fee`, `-setup-tier`, `-run-charge`, `-extra-color-setup`, `-extra-color-run`, `-extra-location-setup`, `-extra-location-run`, `-pms-match`. Variant SKU = `{method_slug}-{charge_kind}-{pricing_hash}`. See **`decoration-pricing.md`** for the full resolution and pricing model.

### `mm-google-shopping.*` (product)

Only present when **Enable Google Merchant Metafields** is on.

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `condition` | `single_line_text_field` | Google condition | `"new"` |
| `google_product_category` | `single_line_text_field` | Taxonomy | `"Apparel & Accessories > Clothing > Shirts & Tops"` |
| `product_type` | `single_line_text_field` | Merchant category | `"T-Shirt"` |
| `brand` | `single_line_text_field` | Brand | `"Nike"` |
| `identifier_exists` | `boolean` | Has GTIN/MPN/brand | `"true"` |
| `adult` | `boolean` | Adult content | `"false"` |
| `age_group` | `single_line_text_field` | `"adult"`, `"kids"`, `"toddler"`, `"infant"`, `"newborn"` | `"adult"` |
| `gender` | `single_line_text_field` | `"male"`, `"female"`, `"unisex"` | `"unisex"` |
| `material` | `single_line_text_field` | Material | `"Cotton"` |
| `pattern` | `single_line_text_field` | Pattern | `"Striped"` |
| `size_type` | `single_line_text_field` | `"regular"`, `"petite"`, `"plus"`, `"tall"`, `"big"`, `"maternity"` | `"regular"` |
| `size_system` | `single_line_text_field` | `"US"`, `"UK"`, `"EU"`, `"DE"`, `"FR"`, `"JP"`, `"CN"`, `"IT"`, `"BR"`, `"MEX"`, `"AU"` | `"US"` |
| `custom_label_0` | `single_line_text_field` | Typically supplier code | `"PCNA"` |
| `custom_label_1` | `single_line_text_field` | Typically line name | `"Premium Apparel"` |
| `custom_label_2` | `single_line_text_field` | Typically main category | `"Shirts"` |
| `custom_label_3` | `single_line_text_field` | Campaign label | user-defined |
| `custom_label_4` | `single_line_text_field` | Campaign label | user-defined |
| `product_detail` | `json` | Structured product details | JSON object |

---

## Variant-level metafields

### `psrestful.*` (variant)

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `part_id` | `single_line_text_field` | Supplier part/variant id (SKU) | `"B03042847"` |
| `variant_gtin` | `single_line_text_field` | GTIN/barcode | `"07501172637351"` |
| `part_price_array` | `json` | Volume tier pricing for the shop's default configuration (decorated when **Default Configuration Type** is Decorated, blank otherwise) | See shape below |
| `blank_part_price_array` | `json` | **Gated.** Same shape as `part_price_array`, but blank (no-decoration) prices. Only on shops with Blank / Sample sales enabled. Always additive to `part_price_array`; both are present on shops that emit it, never overlapping | Same shape as `part_price_array` |
| `sample_price` | `number_integer` | **Gated.** Single-unit blank price in cents. Only written when `blank_part_price_array` has a tier with `quantityMin == 1` (the supplier actually offers qty-1 blank). When absent, the merchant cannot fulfill a single-unit sample; do not infer or fabricate a price | `1500` (= $15.00) |
| `msrp` | `number_decimal` | **Gated, opt-in.** Supplier list price (MSRP) at the part's lowest-quantity tier, in currency units (dollars), not cents. Only on shops with **Save MSRP Metafields** on; absent when the supplier publishes no list price. See "MSRP" below | `"12.50"` |
| `dimension` | `json` | Width, height, depth, weight and their units. Display only; see "Shipping weight" below | `{"dimensionUom": "IN", "width": "3.5", ...}` |
| `apparel_size` | `json` | Style, label size, custom size | See PSRESTful ApparelSize object |
| `specification_array` | `json` | Product specs (length, chest, and so on) | Array of objects |
| `product_package_array` | `json` | Packaging options | Array of objects |
| `shipping_package_array` | `json` | Carton and case-pack configurations (units per carton, gross weight, box dimensions). PromoSync also derives the variant's shipping weight from it; see "Shipping weight" below | Array of objects |
| `color_array` | `json` | Available colors with hex codes, from the PromoStandards `ColorArray`. Separate from color swatches: this is raw per-variant color data, not the swatch mechanism (swatches are option-linked `custom-color-swatch` metaobjects; see "Color swatches") | Array of objects |
| `gallery` | `list.file_reference` | **Gated.** Per-variant media gallery: featured image first, then color-matched extras. See "`gallery` (variant)" below | Array of Shopify File references |

### MSRP: `msrp` (variant) and `msrp_min` (product), gated, opt-in

Written only when the merchant turns on **Save MSRP Metafields** in PromoSync settings (off by default). Shops without the setting do not even get the definitions, so feature-detect by checking the metafield value; never assume.

**Why it exists.** Merchants mix pricing strategies: one supplier is priced cost plus markup, another MSRP minus a discount. Shopify's `compare_at_price` would therefore make every product look on sale, or show nothing. These metafields store the supplier's list price independently of whatever sets the Shopify price, so a theme can show "List price" against "Your price" without touching compare-at.

| Key | Owner | Type | Value |
|-----|-------|------|-------|
| `psrestful.msrp` | variant | `number_decimal` | The supplier's list price at the part's lowest-quantity tier, rounded to 2 decimals. MSRP can differ by size (2XL and up), which is why it is per variant. |
| `psrestful.msrp_min` | product | `number_decimal` | The lowest `msrp` across the product's variants. On Split by Color shops each color product gets its own minimum. |

Rules that matter for themes:

- **Units are currency units (dollars), not cents.** `part_price_array`, `sample_price`, and Liquid's `variant.price` are all in cents. Multiply the MSRP by 100 before comparing or passing to `money`.
- **Absent means "no list price".** PromoSync never derives an MSRP from cost or sell price. Suppliers that publish no list price feed leave both fields empty. Hide the list price line in that case; do not fall back to the Shopify price.
- **It is the list price for the shop's configuration** (Blank or Decorated, after the per-product fallback), read from the supplier's list price feed. Zero-priced padding tiers are ignored.
- **It is not a tier table.** It is one number per variant. Volume pricing still comes from `part_price_array`.
- **It can be below the Shopify price** when the merchant marks up above list. Only show a "List price" strike-through when MSRP is greater than the selling price.
- Both fields are storefront-visible, pinned in the admin, and usable in smart-collection conditions (for example "MSRP (lowest) is greater than 50").
- Kept current by **Update Prices**, so a supplier list-price change flows through without a re-import.

See `integration-patterns.md`, "List price (MSRP) against your price".

### Shipping weight, `dimension`, and `shipping_package_array` (variant)

PromoSync does the shipping-weight work so the theme and the merchant do not have to. On every import and every Add New Variants run, each variant's Shopify inventory item is created with:

- `requiresShipping: true`
- `countryCodeOfOrigin` mapped from the supplier's country of origin (Shopify's code list; `VI` becomes `VG`)
- `measurement.weight`, chosen in this order:
  1. The supplier's per-unit `Dimension.weight`, kept in its own unit when Shopify can represent it (`LB`, `OZ`, `KG`). Milligrams (`ME`) are converted to pounds.
  2. Otherwise, a per-unit weight derived from the largest case pack in `ShippingPackageArray`: carton gross weight divided by units per carton, converted to pounds. The largest pack is used on purpose because it amortizes the empty-box weight best; averaging the packs would bias high. Packs with a zero or missing weight or quantity are skipped.
  3. A derived unit weight above 150 lb is treated as implausible and discarded.
  4. When nothing usable exists, the variant is left without a weight.

Some apparel suppliers publish only carton weights and no per-unit weight, so path 2 is what gives their variants a weight. Hard-goods suppliers usually take path 1.

**What this means for a theme.** Shopify's weight-based shipping rates and carrier-calculated rates work for imported products with no theme code. In Liquid, read the result from `variant.weight` (always grams) and `variant.weight_unit`, and format with `weight_with_unit`. Do not recompute weights from the metafields below; they are there for display (carton counts, product dimensions), not for shipping math. See `integration-patterns.md`, "Shipping weight and carton info".

**`dimension` shape** (the PromoStandards `Dimension` object, one per variant, any field may be null):

```json
{"dimensionUom": "IN", "depth": "3.5", "height": "9.25", "width": "3.5", "weightUom": "LB", "weight": "0.79"}
```

**`shipping_package_array` shape** (one entry per pack configuration the supplier offers; `quantity` is units per carton, `weight` is the gross carton weight):

```json
[
  {"packageType": "Carton", "description": "Master carton", "quantity": "72", "dimensionUom": "IN",
   "depth": "18", "height": "12", "width": "14", "weightUom": "LB", "weight": "31.5"},
  {"packageType": "Carton", "description": "Inner pack", "quantity": "12", "dimensionUom": "IN",
   "depth": "9", "height": "6", "width": "7", "weightUom": "LB", "weight": "5.6"}
]
```

Numbers are serialized as **strings** (they are decimals upstream). In Liquid, coerce with `| plus: 0` before comparing or sorting. `packageType` is required by the PromoStandards spec but a few suppliers send null. `weightUom` is one of `LB`, `OZ`, `KG`, `ME` (milligram); `dimensionUom` is one of `IN`, `FT`, `YD`, `MM`, `CM`, `MR` (meter).

### `gallery` (variant): gated, may be absent

The per-variant gallery is written only when **both** of these hold:

1. The shop's PromoSync plan includes the per-variant media gallery (Trial, Business and Enterprise plans; see the PromoSync docs for current plan eligibility).
2. **Variant Grouping** is **One Product (All Variants)**. Split by Color shops have one product per color, so each product's native media gallery already covers it and a per-variant gallery would be redundant.

When the gate is closed, the metafield is not written at all: `variant.metafields.psrestful.gallery` is empty. Themes must check `.value.size > 0` before relying on it and fall back to `product.media` plus `variant.featured_media`.

**Shape:** `list.file_reference`, a Liquid array of Shopify File objects. Each entry is typically a `MediaImage` (use `.image` to get the underlying Image object); occasionally a generic file (use `.url`). Order is meaningful:
- Index 0 = the variant's featured image (same URL as `variant.featured_media`)
- Index 1+ = additional color-matched media, deduplicated against the featured image

**Liquid access:**
```liquid
{%- assign vg = variant.metafields.psrestful.gallery.value -%}
{%- if vg.size > 0 -%}
  {%- for media in vg -%}
    {%- if media.image -%}
      <img src="{{ media.image | image_url: width: 1200 }}" alt="{{ media.image.alt | escape }}">
    {%- endif -%}
  {%- endfor -%}
{%- endif -%}
```

See `integration-patterns.md`, "Variant gallery (gated)", for the full pattern including fallback and JS variant switching.

### How PromoSync matches supplier media to variants

Suppliers tag their media inconsistently: some tag every asset with a color, some leave color blank but always set a part id, some send only product-level media. To build the per-variant `gallery` list, PromoSync collects media for each variant in this order:

1. Media tagged with the variant's part id. If any, use it.
2. Otherwise, media tagged with the variant's primary color (the **Color Field** setting: `colorName` by default, or `standardColorName`).
3. Otherwise, media tagged with the other color field.
4. Deduplicate URLs while preserving order.

For suppliers whose media is tagged by part id only (HIT Promotional Products today), PromoSync matches by part id alone and never falls back to color, so one color's images are never attached to another variant.

Implications for theme work:

- **Gallery sizes vary by supplier.** Part-id-only suppliers often yield one or two entries per variant; suppliers that tag colors well often yield five to ten per color.
- **An empty gallery list is not necessarily a bug.** It can mean the supplier did not tag any media against this part or its color. The theme's job is to fall back to `product.media` gracefully (see the integration pattern).
- **Order reflects the matching order**, with the featured image hoisted to index 0. Don't expect a curated sequence; index 0 is reliable, index 1 and later are supplier-driven.

### Blank / sample metafields (`blank_minimum_quantity`, `blank_part_price_array`, `sample_price`): gated trio

These three metafields are present only when **both** of these hold:

1. **Enable Blank / Sample Sales** is on for the shop. It is not in the merchant-facing settings by default; PSRESTful support turns it on for a shop on request.
2. **Default Configuration Type** is **Decorated**, so the shop sells decorated products by default and the regular `minimum_quantity` and `part_price_array` hold the decorated values.

The point is to let a Decorated-default shop also surface blank pricing and single-unit sample pricing on the storefront, so customers can choose between **Decorated**, **Blank**, and **Sample** purchase modes on the same product.

**Why not on Blank-default shops?** On a Blank-default shop, the regular `minimum_quantity` and `part_price_array` already are the blank values; emitting `blank_minimum_quantity` and `blank_part_price_array` would be the same data twice.

**Key invariants:**

- `blank_part_price_array` has the exact same JSON shape as `part_price_array` (`[{quantityMin, price-in-cents}, …]`, ascending by `quantityMin`, last match wins). Reuse the same matching logic; just swap the metafield path.
- `blank_minimum_quantity` and `blank_part_price_array` are written independently per product and variant. A product with no blank pricing data from the supplier won't get them even when the gate is open. Themes must `if metafield.value.size > 0` / `if metafield`.
- `sample_price` is **derived**, not configured. It is the lowest blank tier's price, written only when that tier's `quantityMin == 1`. If the supplier's blank tiers start at qty 6, `sample_price` is omitted; selling a single sample isn't fulfillable.
- `sample_price` is in cents, like all PromoSync price fields. Render with `money` directly.
- The decorated values stay on `minimum_quantity` / `part_price_array`; there is no `decorated_part_price_array`. So a Decorated-default shop with the gate open has: `part_price_array` = decorated tiers, `blank_part_price_array` = blank tiers, `sample_price` = qty-1 blank shortcut.
- **At checkout, Blank and Sample lines are charged at the variant's Shopify price today.** The Quantity Price Breaks Function reads only `part_price_array`. See `integration-patterns.md`, "Blank / Decorated / Sample purchase flows", before offering these modes on a storefront that takes payment.

**Liquid access:**

```liquid
{%- comment -%} Decorated (default) {%- endcomment -%}
{%- assign dec_min = product.metafields.psrestful.minimum_quantity -%}
{%- assign dec_tiers = variant.metafields.psrestful.part_price_array.value -%}

{%- comment -%} Blank: gated, may be nil {%- endcomment -%}
{%- assign blank_min = product.metafields.psrestful.blank_minimum_quantity -%}
{%- assign blank_tiers = variant.metafields.psrestful.blank_part_price_array.value -%}

{%- comment -%} Sample: gated AND requires a qty-1 blank tier; may be nil even when blank_tiers exists {%- endcomment -%}
{%- assign sample_cents = variant.metafields.psrestful.sample_price -%}
```

### `part_price_array` JSON shape

```json
[
  { "quantityMin": 1,   "price": 2500 },
  { "quantityMin": 10,  "price": 2200 },
  { "quantityMin": 25,  "price": 2000 },
  { "quantityMin": 50,  "price": 1800 },
  { "quantityMin": 100, "price": 1500 }
]
```

**Critical rules:**
- `price` is in **cents** in the store's currency (2500 = $25.00 on a USD store)
- Tiers are sorted by `quantityMin` ascending
- **Match by iterating all tiers and taking the last one where `quantity >= quantityMin`**
- Use `{{ tier.price | money }}`; do not divide by 100 first and do not hardcode a currency symbol
- The quantity to match is the cart line's own quantity in **Per variant** mode, or the sum of every cart line of the same product in **Combined by product** mode (PromoSync > Shop Settings > Volume Pricing > Mode). See `integration-patterns.md`, "Match the store's volume pricing mode"
- The Quantity Price Breaks Shopify Function reads this metafield and applies the actual discount at checkout when **Enable Volume Pricing** is on

### `mm-google-shopping.*` (variant)

Only present when **Enable Google Merchant Metafields** is on.

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `gtin` | `single_line_text_field` | Variant GTIN/barcode | `"07501172637351"` |
| `mpn` | `single_line_text_field` | Manufacturer part number | `"B03042847"` |
| `title` | `single_line_text_field` | Google variant title | `"Premium T-Shirt - Black - Large"` |
| `cost_of_goods_sold` | `single_line_text_field` | COGS with currency | `"6.04 USD"` |
| `color` | `single_line_text_field` | Color name | `"Black"`, `"Monterey Sage"` |
| `size` | `single_line_text_field` | Size | `"Large"`, `"2XL"` |
| `item_group_id` | `single_line_text_field` | Group id for variant set | `"product-123-group"` |
| `age_group` | `single_line_text_field` | Age group (if shop opts in) | `"adult"` |
| `gender` | `single_line_text_field` | Gender (if shop opts in) | `"unisex"` |

---

## Color swatches (option-linked metaobjects)

PromoSync renders **native** Shopify color swatches by linking a product's **Color option** to **metaobjects**, not by writing a value the theme parses ad hoc.

### The two definitions (created once per shop)

1. **Metaobject definition**: type `custom-color-swatch`, name "Color Swatch", storefront access public read. Fields:

   | Field key | Type | Holds |
   |---|---|---|
   | `label` | `single_line_text_field` | Color name, for example `"Red"` |
   | `hex_color` | `single_line_text_field` | Hex, for example `"#FF0000"` |

2. **Product metafield definition**: namespace `psrestful`, key `color-swatches`, **type `list.metaobject_reference`**, owner type `PRODUCT`, validated against the metaobject definition above, storefront access public read.

### Per-color metaobjects

For each color that has a known hex, PromoSync creates or updates one metaobject: type `custom-color-swatch`, handle = slugified color name, fields `label` = color name and `hex_color` = hex. Colors without a hex are skipped: no metaobject, no swatch.

### Where the hex comes from

PSRESTful's normalized color data for the supplier product: each color name maps to a hex and a color group. When PSRESTful has no hex for a color, that color gets no swatch.

### The actual link (what makes swatches render)

The Color option is created **option-linked**. The option payload PromoSync sends to Shopify looks like:

```json
{
  "name": "Color",
  "position": 1,
  "linkedMetafield": { "namespace": "psrestful", "key": "color-swatches" },
  "values": [
    { "name": "Red", "linkedMetafieldValue": "gid://shopify/Metaobject/123" }
  ]
}
```

Each **option value** carries `linkedMetafieldValue` = the metaobject GID. With no metaobject for a color, the value degrades to plain `{ "name": "Red" }` (no swatch).

### The Color Field setting

**Color Field** (`colorName` by default, or `standardColorName`) selects which PromoStandards color field names the variant's color. PromoSync falls back to the other field, then to the part's primary color.

### Gating

Swatches are written when **Enable Color Swatches** is on in PromoSync settings, or automatically on plans that include the feature (Trial, Business and Enterprise). When off, the Color option is unlinked and no swatch metaobjects or metafield are written.

### Adding variants later

**Add New Variants** reuses the shop's existing swatch metaobjects. If a new variant's color has no swatch metaobject, that variant is not added and the result email reports that the color is not linked to any metaobject and that the merchant must add a swatch for this color and retry. A missing swatch blocks the variant add; it is not skipped silently.

### Reading swatches in a theme (Liquid)

`color-swatches` is `list.metaobject_reference`, so `.value` is an **array** of `custom-color-swatch` metaobjects, each exposing `.label` and `.hex_color`.

> **Hyphenated key gotcha.** `product.metafields.psrestful.color-swatches` parses the hyphen as subtraction. You **must** use bracket access: `product.metafields.psrestful["color-swatches"].value`.

```liquid
{%- assign swatches = product.metafields.psrestful["color-swatches"].value -%}
{%- for swatch in swatches -%}
  {{ swatch.label }}: {{ swatch.hex_color }}
{%- endfor -%}
```

Length via `…["color-swatches"].value.count` (reference lists use `count`, not `size`). Because the metaobject is a **custom** type (not Shopify's standard color or pattern metaobject), `option_value.swatch` is **not** auto-populated; a custom theme should read this list and match `swatch.label` to the option value name. See `integration-patterns.md`, "Color swatches", for the full render snippet.

---

## Configurable products (multi-part-group: kits, sets, bundles)

Enabled with **Enable Configurable Products** in PromoSync > Shop Settings (off by default). A small but growing class of supplier products are *configurable*: the supplier's Product Pricing and Configuration (PPC) data describes them as several **part groups** (a plush bear plus an apparel item, a tumbler plus a color-matched straw, a six-piece new-hire kit). When the setting is on **and** the product has more than one part group, PromoSync builds the **real variant matrix** from the part-group structure instead of flattening every part into one option. Regular single-group products are unaffected.

To the theme, a configurable product is just a normal multi-option Shopify product; read its metafields exactly as you would any other. The differences worth knowing:

- **Option names can be generic.** Each part-group axis becomes a Shopify option. When the supplier's group label is the generic `"Part Group"`, the option name falls back to `Option 1`, `Option 2`, and so on; otherwise it's the supplier label (for example `SUNGLASSES`). **Don't assume an option is named `Color`.**
- **Option values may be disambiguated.** When one axis mixes sub-types (the apparel axis spans SHIRT / HOODIE / BANDANA, each with a "NAVY BLUE"), values are prefixed to stay unique: `SHIRT - NAVY BLUE`, `HOODIE - NAVY BLUE`. Render them as-is.
- **SKU is a joined bill of materials.** A variant's SKU concatenates every component part id with `-`, for example `1200BRN-1210BLK` (bear + shirt) or `5306RBLK-5306RBLKSTRAW` (tumbler + straw). **Never parse the SKU to recover a part id**; use the `psrestful.part_id` metafield.
- **`psrestful.part_id` is the priced component, not the whole SKU.** Each configurable variant's `part_id` is the priced part of the combination (the apparel for the bear, the body for the tumbler, the anchor for a kit). Per-part lookups (inventory, decoration enrichment) key off this, not the joined SKU.
- **`part_price_array` works exactly as normal**: the variant's tier array is the priced component's tiers (cents, ascending). Tier-pricing display, the Quantity Price Breaks Function, and `minimum_quantity` need no special handling.
- **`location_decorations` is the union across the combination's parts.** A bear+shirt variant exposes both the bear's and the shirt's decoration locations; a tumbler exposes the body's locations (the straw has none). Decoration pricing reads it the same way.
- **Variant images** are resolved per variant from the best-matching component (exact part id, then shared color code, then product group image), so `variant.featured_media` is set normally and the `gallery` gating rules are unchanged.

**Bottom line for theme work:** treat configurable products like any multi-option product. The only practical gotchas are (a) option names may be `Option N`, (b) option values may carry a `TYPE - COLOR` prefix, and (c) the SKU is a joined bill of materials, so always prefer `psrestful.part_id` over the SKU.

---

## PromoSync settings that control metafields

These are the PromoSync settings (PromoSync > Shop Settings unless noted) that decide which metafields get written. If a metafield is missing on a store, check the setting before the theme. Names are as the merchant sees them.

| Setting | Effect on metafields |
|---------|----------------------|
| **Enable Google Merchant Metafields** | Writes `mm-google-shopping.*` on products and variants. On some plans this is on automatically. |
| **Enable Color Swatches** (automatic on Trial, Business and Enterprise plans) | Creates the `custom-color-swatch` metaobjects and the `psrestful.color-swatches` definition, and links the Color option's values to per-color metaobjects. See "Color swatches". |
| **Color Field** (`colorName` by default, or `standardColorName`) | Which PromoStandards color field names a variant's color, with the other as fallback. Affects swatch labels and media matching. |
| **Variant Grouping** (One Product (All Variants), or Split by Color) | Split by Color creates one product per color family. Also decides whether `psrestful.gallery` is written (only with One Product). |
| **Default Configuration Type** (Blank or Decorated) | Drives what `minimum_quantity` and `part_price_array` mean, and which supplier price set feeds the Shopify price. |
| **Enable Blank / Sample Sales** (turned on by PSRESTful support on request) | With Default Configuration Type = Decorated, writes `blank_minimum_quantity`, `blank_part_price_array` and `sample_price` so the storefront can sell Decorated, Blank and Sample versions of a product. See "Blank / sample metafields". |
| **Save MSRP Metafields** | Writes `psrestful.msrp` (variant) and `psrestful.msrp_min` (product) on import and keeps them current on Update Prices. See "MSRP". |
| **Import Minimum Quantity as 1** | Forces `psrestful.minimum_quantity = 1` regardless of the supplier value. |
| **Enable Default Tier Pricing** plus **Tier Pricing Rules** (PromoSync > Settings) | Writes `psrestful.part_price_array` from the merchant's own rules for products whose tags match a rule, in place of or in addition to supplier tiers. |
| **Enable Default Location Decorations** plus **Location Rules** | Augments `psrestful.location_decorations` with default locations and methods for products whose tags match a rule. |
| **Enable Decoration Tags** (ask PSRESTful support if it is not in your settings) | Tags imported products with their decoration methods, which is handy for collections and filters. |
| **Enable Configurable Products** | Builds the real variant matrix for products with more than one part group. See "Configurable products". |
| **Enable Volume Pricing** and **Mode** (Per variant or Combined by product), under Volume Pricing | Creates and configures the Quantity Price Breaks automatic discount that applies `part_price_array` tiers at checkout. See `theme-blocks.md`. |
| PromoSync plan (Trial, Business, Enterprise) | Gates `psrestful.gallery`, and turns on color swatches and Google Merchant metafields without the setting. Plan eligibility is listed in the PromoSync docs. |
| `psrestful.extra_id` (a product metafield, not a setting) | PromoSync's id for the product. Required for Location Decorations Enrichment; without it, no `*VariantId` fields are written to `location_decorations` and decorations stay unpriced. See `decoration-pricing.md`. |

---

## Which PromoSync job writes what

| Job (PromoSync) | Writes |
|-----------------|--------|
| Product import | Every `psrestful.*` and `mm-google-shopping.*` metafield on the new product and its variants, the color swatch metaobjects and Color option links, and the inventory item's weight and country of origin |
| Update Metafields | Refreshes the same metafields on existing products (including `marketing_points` and rule-based `part_price_array`); the fix for products imported before a metafield existed |
| Update Prices | Shopify prices and costs, `part_price_array`, `msrp`, `msrp_min` |
| Add New Variants | Metafields on the new variants; needs a swatch metaobject for any new color when swatches are on |
| Insert Decorations | The canonical decoration products: `method_slug`, `charge_kind`, `default_variant_id` on the product; `pricing_hash`, `suppliers`, `part_price_array` (run charge) and `setup_tier_array` (setup tier) on the variants |
| Location Decorations Enrichment | The `*VariantId` fields inside `psrestful.location_decorations` (runs after Insert Decorations) |
| Add Missing Images | Product media and variant image links from supplier media |

## Read paths (who reads each metafield)

| Reader | Keys |
|--------|------|
| Theme app block **Tier Pricing** | `psrestful.part_price_array` |
| Theme app block **Location Decorations** | `psrestful.location_decorations` |
| Theme app block **Minimum Quantity Notice** | `psrestful.minimum_quantity` |
| Theme app embed **Pricing Engine** | `psrestful.minimum_quantity`, `psrestful.part_price_array` |
| Theme app block **Volume Pricing Note** | `psrestful.part_price_array` |
| Shopify Function **Quantity Price Breaks** | `psrestful.part_price_array` on every cart line, decoration run-charge lines included |
| Update Metafields (variant matching) | `psrestful.part_id`, `psrestful.extra_id` |
| Product import (dedupe) | `psrestful.product_id`, `psrestful.supplier_code` |
| Theme gallery integrations (custom) | `psrestful.gallery` (variant, gated; falls back to `product.media` when absent) |
| Theme product-specs or features tab (custom) | `psrestful.marketing_points` (product, raw supplier copy, filter with an allow-list of `pointType`s) |
| Native Shopify variant picker and theme swatch UI | Color option's `linkedMetafield` to `psrestful.color-swatches` metaobjects (`label`, `hex_color`). Custom themes read `product.metafields.psrestful["color-swatches"].value` directly |
| Theme Blank/Sample purchase UI (custom) | `psrestful.blank_minimum_quantity`, `psrestful.blank_part_price_array`, `psrestful.sample_price` (gated trio) |
| Theme list-price display (custom) | `psrestful.msrp` (variant) and `psrestful.msrp_min` (product); gated, opt-in; decimal in currency units, multiply by 100 before comparing with `variant.price` |
| Theme decoration-pricing UI (custom) | Enriched `psrestful.location_decorations` `*VariantId` fields plus the canonical products' `psrestful.default_variant_id`, variant `psrestful.part_price_array` and `psrestful.setup_tier_array`. Not done by the shipped block; see `decoration-pricing.md` |

---

## Example payloads (for GraphQL `metafieldsSet`)

### Variant tier pricing
```json
{
  "ownerId": "gid://shopify/ProductVariant/789012",
  "namespace": "psrestful",
  "key": "part_price_array",
  "type": "json",
  "value": "[{\"quantityMin\":50,\"price\":1250},{\"quantityMin\":100,\"price\":1100}]"
}
```

### Product location decorations (unenriched write shape)
```json
{
  "ownerId": "gid://shopify/Product/123456",
  "namespace": "psrestful",
  "key": "location_decorations",
  "type": "json",
  "value": "[{\"locationName\":\"Front Chest\",\"decorations\":[{\"decorationName\":\"Screen Print\",\"maxImprintColors\":10,\"default\":true,\"priceIncludes\":true}]}]"
}
```

Location Decorations Enrichment later adds the `*VariantId` fields (and `locationId`, `decorationsIncluded`, and so on) to this same metafield; see the enriched shape above and `decoration-pricing.md`.

### Product minimum quantity
```json
{
  "ownerId": "gid://shopify/Product/123456",
  "namespace": "psrestful",
  "key": "minimum_quantity",
  "type": "number_integer",
  "value": "25"
}
```
