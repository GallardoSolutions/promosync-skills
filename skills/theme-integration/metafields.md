# PromoSync Metafield Catalog

Authoritative, hardcoded catalog of every Shopify metafield PromoSync writes, reads, or expects. Last refreshed: 2026-04-20. When something is missing here, check https://promosync-docs.psrestful.com.

Source of truth: the metafield definitions registered by the PromoSync app.

---

## Namespaces

| Namespace | Owner types | When written |
|-----------|-------------|--------------|
| `psrestful` | Product, ProductVariant | Always — core sync |
| `mm-google-shopping` | Product, ProductVariant | Only when `Shop.enable_google_merchant=true` |
| `my_fields` | Product, ProductVariant | Only when `Shop.is_hit_a_double=true` |

---

## Product-level metafields

### `psrestful.*` (product)

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `supplier_code` | `single_line_text_field` | Supplier code | `"PCNA"`, `"SanMar"` |
| `product_id` | `single_line_text_field` | Supplier's product ID | `"B03042000"` |
| `extra_id` | `number_integer` | PromoSync internal product id | `12345` |
| `status` | `single_line_text_field` | Lifecycle status | `"active"`, `"draft"`, `"closeout"`, `"discontinued"` |
| `effective_date` | `date_time` | When product becomes available | `"2025-01-15T00:00:00"` |
| `export` | `boolean` | Export flag | `"true"` / `"false"` |
| `primary_material` | `single_line_text_field` | Main material | `"100% Polyester"` |
| `brand` | `single_line_text_field` | Brand | `"Nike"` |
| `country_of_origin` | `single_line_text_field` | ISO country | `"MX"`, `"USA"` |
| `is_caution` | `boolean` | Has caution warning | `"true"` / `"false"` |
| `caution_comment` | `multi_line_text_field` | Caution text | `"Contains latex."` |
| `line_name` | `single_line_text_field` | Product line | `"Premium Apparel"` |
| `minimum_quantity` | `number_integer` | Minimum order qty for the default configuration (decorated when `default_configuration_type='Decorated'`, blank otherwise) | `5`, `25`, `100` |
| `blank_minimum_quantity` | `number_integer` | **Gated.** Minimum order qty for the *blank* (undecorated) configuration. Only emitted when `Shop.emit_blank_sample_metafields == true`. | `1`, `6`, `12` |
| `lead_time` | `number_integer` | Lead time in days | `14` |
| `is_on_demand` | `boolean` | On-demand product | `"true"` / `"false"` |
| `is_rush_service` | `boolean` | Rush service available | `"true"` / `"false"` |
| `imprint_size` | `single_line_text_field` | Imprint area size | `"1.5\" x 1.5\""` |
| `price_expires_date` | `date_time` | Price expiry | `"2025-12-31T23:59:59"` |
| `is_hazmat` | `boolean` | Hazmat flag | `"true"` / `"false"` / null |
| `default_set_up_charge` | `single_line_text_field` | Setup fee | `"$50"` or `"50"` |
| `default_run_charge` | `single_line_text_field` | Per-unit run charge | `"$0.25/unit"` |
| `unspsc_commodity_code` | `number_integer` | UNSPSC code | `23101500` |
| `location_decorations` | `json` | Decoration locations + methods | See shape below |
| `features` | `multi_line_text_field` | Feature bullets | `"Water resistant\nUV protected"` |
| `care_instructions` | `multi_line_text_field` | Care instructions | `"Machine wash warm"` |
| `charge_type` | `single_line_text_field` | Charge category | `"Setup"`, `"Run Charge"`, `"Order"` |
| `marketing_points` | `json` | Supplier `ProductMarketingPointArray` as-is: `[{pointType, pointCopy}]`. **Not normalized** — every supplier labels points differently and some send internal/distributor data. Only written when the supplier sends at least one point with copy. Never render unfiltered — see "Marketing points" below. | `[{"pointType":"Bullet Point","pointCopy":"8 Panels"}]` |
| `color-swatches` | `list.metaobject_reference` | **Option-linked** list of `custom-color-swatch` metaobjects (each has `label` + `hex_color`) that power native Color-option swatches. See "Color swatches" below — type is a *list*, not a scalar. | `[gid://shopify/Metaobject/123, …]` |

### `location_decorations` JSON shape

This metafield exists in **two states**, both valid:

- **Unenriched (simple)** — what PromoSync writes on import / default application. Just locations and
  method names. Use it for display-only decoration selectors.
- **Enriched (v3)** — after PromoSync's decoration **enricher** runs, each decoration entry gains
  `*VariantId` fields pointing at canonical decoration-product variants, plus richer location fields
  (`locationId`, `decorationsIncluded`, `minDecoration`, `maxDecoration`, `locationRank`). This is what
  lets a theme **price** decorations (setup / run / extra-color / extra-location / PMS). It's the **same
  metafield** — enrichment adds fields in place.

**Enriched example** (one location, two decorations):

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
        "setupFeeVariantId": "gid://shopify/ProductVariant/66769244782637",
        "runChargeVariantId": "gid://shopify/ProductVariant/66769247698989",
        "extraColorSetupVariantId": "gid://shopify/ProductVariant/66769250000111",
        "extraColorRunVariantId": "gid://shopify/ProductVariant/66769250000222",
        "extraLocationSetupVariantId": "gid://shopify/ProductVariant/66769252220973",
        "extraLocationRunVariantId": "gid://shopify/ProductVariant/66769252220988",
        "pmsMatchVariantId": "gid://shopify/ProductVariant/66769252450349"
      },
      {
        "decorationId": 58350,
        "decorationName": "Embroidery",
        "maxImprintColors": 6,
        "default": false,
        "priceIncludes": false,
        "setupFeeVariantId": "gid://shopify/ProductVariant/66769262051373",
        "runChargeVariantId": "gid://shopify/ProductVariant/66769269784621"
      }
    ]
  }
]
```

Location-entry fields:
- `locationName` — user-facing label (e.g., "Front Chest", "Left Sleeve"). May have trailing whitespace — trim it.
- `locationId` — supplier's location id; internal, don't surface to customers.
- `decorationsIncluded` — decorations included before extra-location charges apply (feeds the extra-location math).
- `defaultLocation` — default-select this location in the UI when `true`.
- `minDecoration` / `maxDecoration` — min/max decorations selectable here. `maxDecoration: 0` → skip rendering.
- `locationRank` — display order (lower = earlier).
- `decorations[]` — array of methods available at that location.

Per-decoration fields:
- `decorationName` — **canonical** method name ("Screen Print", "Embroidery", "Laser Engraving", "Full Color", etc.), already alias-resolved server-side.
- `decorationId` — supplier's decoration id; internal, don't surface.
- `maxImprintColors` — max distinct imprint colors supported (color-count UI ceiling).
- `default` — whether this decoration is pre-selected.
- `priceIncludes` — base product price already includes this decoration (skip setup/run charges when `true`).
- `setupFeeVariantId`, `runChargeVariantId`, `extraColorSetupVariantId`, `extraColorRunVariantId`, `extraLocationSetupVariantId`, `extraLocationRunVariantId`, `pmsMatchVariantId` — **enrichment-only** GIDs of canonical decoration-product variants. An **absent** field means "this charge doesn't apply" (not "missing data"). camelCase exactly as written.

> The enricher **strips** legacy fields on every run: `*ProductId` (v2 product pointers) and the pre-split
> `extraColorVariantId` / `extraLocationVariantId` (replaced by the `*Setup`/`*Run` pairs). Don't read them.

**For the full decoration-pricing model** — canonical products, the 7 charge kinds, variant resolution
(`default_variant_id` fallback), the setup/run/extra/PMS math, and multi-line cart-add — see
**`decoration-pricing.md`**.

### `psrestful.*` (Liquid access)

```liquid
{{ product.metafields.psrestful.minimum_quantity }}
{{ product.metafields.psrestful.location_decorations.value }}  {% comment %} array of objects {% endcomment %}
{{ product.metafields.psrestful.supplier_code }}
```

### `marketing_points` (product) — raw supplier copy, filter before display

Added 2026-09-30. Hand-verified 2026-10-02.

**What it is.** The PromoStandards Product Data `ProductMarketingPointArray`, copied as
JSON in its PromoStandards shape. Suppliers like Peerless put most of their spec sheet here
(bullets, material, dimensions) instead of the part-level `specification_array`.

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
- Product-level, type `json`, **no shop/plan gate**. Points whose `pointCopy` is empty are
  dropped and `pointCopy` is trimmed. If nothing is left, the metafield is **not written** (absent ≠ bug).
- `pointType` is passed through untouched. It can be **`null`** (Showdown Displays) or
  free text.
- Written on **import** and by **Update Metafields**. Products imported before 2026-09-30 only
  get it after the merchant runs Update Metafields on them.
- The definition is **not** storefront-exposed (`storefront_access=False`). Liquid can
  read it; Storefront-API JS (`node(id:)`, `productByHandle`) cannot.
- PromoSync does **no** filtering, mapping or normalization. Deciding what to show is
  theme work.

**How real suppliers use it** (survey of one live store's ~12k imported products, 40 suppliers,
2026-10-02, from PSRESTful `extra_product.marketing_points`, which is exactly what PromoSync writes):

| Supplier (code) | `pointType` values seen | Display verdict |
|---|---|---|
| Peerless, Stromberg (`STROMBERGBRAND`), Diamondback (`DIAMONDBACKBRANDING`), WOWLine (`WOW`), SugarSpot, Raining Rose, Coloring Book Solutions, most numeric-code suppliers | `Bullet Point` + `Additional Info`, `Material`, `Product Width/Length/Height (IN)`, `Product Weight (LB)`, 0/1 flags (`Eco-Friendly`, `Woman Owned`, `PromoCares Collection`, `QCA Certified Supplier`), `Industry`, `Must Be Ordered In Increments Of` | Show `Bullet Point`; optionally measurements/material as label: value rows; hide the rest |
| Primeline (`PRIME`), Ariel, Gemline (`GEM`) | `Feature` (GEM also `ECOCertification`, `CarbonFootprint`, `Product`) | Show `Feature` |
| HIT | `Highlight` (only ~10% of HIT products have any) | Show `Highlight` |
| Koozie Group (`BIC`) | `BTH` only (~4–5 per product). Verified 2026-10-02: real product bullets ("Snap-on slider lid", "80 sheets", "Assembled in USA") | **Show** as bullets. Skip the ~3% that start with `Top markets:` (target-market notes for distributors). A few mention option charges ("Optional magnet available for an additional charge") |
| Logomark (`LMRK`) | `Feature`, `Quote`, `PROP 65` (8 of ~600 products) | Show `Feature` |
| Showdown Displays (`sign`) | `pointType` = `null`, on about 1 in 3 products. Verified 2026-10-02: always **exactly one** point, a ~100-char marketing tagline ("Shade has never been this simple. Our patented E-Z Glide lever…") | **Show** as a tagline/intro line, not a bullet list |
| TerryTown (`terry`) | The copy itself is stored in `pointType` | Malformed: skip |
| Starline (`STAR`) | `Packaging`, `Production Time`, `Price`, `Material`, `NOTE`, `Output`, … (20 types) | Mostly distributor notes; hide `Price`/`NOTE`/`Production Time` |
| Snugz (`SNUGZ`) | `Compliance` only | Hide (or a dedicated compliance block) |
| Goldstar | `HTS US`, `HTS CA`, `HTS EU` only (customs codes) | Hide |
| PCNA | ~98 per product, all internal: `PCNAAttributes-CanonicalURL`, `PCNAAttributes-GlobalCategory`, `PCNAInternal-MadeInUSA`, … (~90k distinct types) | **Hide everything.** Not product copy |
| NuPromo, Apli | `Additional Info` (+ `Shipping`) | Hide (distributor notes) |
| BEL Promo, The Magnet Group (`MAG`), Gold Bond (`Gold`), Evans, Tekweld, High Caliber, Hirsch, Compass, ETS Express, Lincoln Line, Illini, Galaxy Balloon | — (send nothing) | Metafield absent |

**Takeaways for theme work.**
1. **Allow-list, never deny-list.** Show only known customer-facing types; everything else stays
   hidden by default. A deny-list would leak PCNA's ~98 internal attributes.
2. **Match several bullet names.** A `Bullet Point`-only filter renders nothing for HIT,
   Primeline, Ariel, Gemline and Koozie. A reasonable starting allow-list is
   `Bullet Point`, `Feature`, `Highlight`, `BTH`. Untyped (`null`) points are a single
   tagline (Showdown), so render them separately as an intro line.
3. **0/1 flags read wrong on a PDP.** `Eco-Friendly: 0` means *not* eco-friendly. Never print
   flag types as text.
4. **`Additional Info` is usually a distributor note** (pricing/decoration terms that may contradict
   the store's own decoration pricing). Hide it by default.
5. **Expect absence.** Many suppliers send nothing, others only for some products. Always
   `if points` / `points.size > 0` and render nothing (or fall back to `product.description`).
6. Supplier usage drifts. Re-check a couple of products per supplier when adding a supplier.
   PSRESTful's Data Explorer shows the raw `ProductMarketingPointArray` per product.

Pattern: `integration-patterns.md` → "Marketing points (product specs)".

### Canonical decoration products (`psrestful.*`)

PromoSync's "Insert decorations" action creates a **shop-wide catalog** of canonical decoration products —
one Shopify product per **(imprint method × charge kind)**, vendor `PromoSync`, tagged
`[<charge_kind>, promosync_decoration]`. They hold the actual decoration prices; physical products point
at their variants via the `*VariantId` fields on `location_decorations` (above). These are **separate
Shopify products**, not metafields on the physical product.

**Product-level metafields** (namespace `psrestful`):

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `method_slug` | `single_line_text_field` | Canonical imprint-method slug (lookup/grouping key) | `"screen-print"` |
| `charge_kind` | `single_line_text_field` | One of `setup_fee`, `run_charge`, `extra_color_setup`, `extra_color_run`, `extra_location_setup`, `extra_location_run`, `pms_match` | `"run_charge"` |
| `default_variant_id` | `variant_reference` | **Storefront fallback** variant for products with no per-product `*VariantId` (e.g. SanMar / S&S blanks). Auto-picked as the highest-product-count pricing class; merchant can pin a different one. | `gid://shopify/ProductVariant/667…` |

**Variant-level metafields** (one variant per pricing equivalence class; namespace `psrestful`):

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `pricing_hash` | `single_line_text_field` | 8-char hash identifying this pricing class across re-runs; also the SKU suffix | `"a1b2c3d4"` |
| `suppliers` | `json` | Supplier codes sharing this class (PromoSync's matching key — theme rarely reads it) | `["ARIEL","HIT"]` |
| `part_price_array` | `json` | **Run-charge variants only.** Quantity-tier ladder, `price` in **integer cents**, ascending, equal-price tiers collapsed | `[{"quantityMin":1,"price":85},{"quantityMin":144,"price":70}]` |

Handle = `slugify(decorationName) + suffix` where suffix is `-setup-fee`, `-run-charge`,
`-extra-color-setup`, `-extra-color-run`, `-extra-location-setup`, `-extra-location-run`, `-pms-match`.
Variant SKU = `{method_slug}-{charge_kind}-{pricing_hash}`. See **`decoration-pricing.md`** for the full
resolution + pricing model.

### `mm-google-shopping.*` (product)

Only present when `Shop.enable_google_merchant=true`.

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

### `my_fields.*` (product) — Hit a Double multi-location inventory

Only present when `Shop.is_hit_a_double=true`.

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `ds_style` | `single_line_text_field` | DropShip style/product id | `"DS12345"` |
| `sm_style` | `single_line_text_field` | SanMar style id | `"SM9876"` |
| `ss_style` | `single_line_text_field` | S&S Activewear style id | `"SS5432"` |
| `cm_style` | `single_line_text_field` | Carolina Made style id | `"CM1111"` |

---

## Variant-level metafields

### `psrestful.*` (variant)

| Key | Type | Purpose | Example |
|-----|------|---------|---------|
| `part_id` | `single_line_text_field` | Supplier part/variant id (SKU) | `"B03042847"` |
| `variant_gtin` | `single_line_text_field` | GTIN/barcode | `"07501172637351"` |
| `part_price_array` | `json` | Volume/tier pricing for the default configuration (decorated when `default_configuration_type='Decorated'`, blank otherwise) | See shape below |
| `blank_part_price_array` | `json` | **Gated.** Same shape as `part_price_array`, but blank (no-decoration) prices. Only emitted when `Shop.emit_blank_sample_metafields == true`. Always *additive* to `part_price_array` — both are present on shops that emit it, never overlap. | Same shape as `part_price_array` |
| `sample_price` | `number_integer` | **Gated.** Single-unit blank price **in cents**. Only emitted when `blank_part_price_array` has a tier with `quantityMin == 1` (i.e. the supplier actually offers qty-1 blank). When absent, the merchant cannot fulfill a single-unit sample — do not infer or fabricate a price. | `1500` (=$15.00) |
| `dimension` | `json` | Width/height/depth/weight + UOMs | See PSRESTful Dimension object |
| `apparel_size` | `json` | Style, label size, custom size | See PSRESTful ApparelSize object |
| `specification_array` | `json` | Product specs (length, chest, etc.) | Array of objects |
| `product_package_array` | `json` | Packaging options | Array of objects |
| `shipping_package_array` | `json` | Master carton / shipping packaging | Array of objects |
| `color_array` | `json` | Available colors with hex codes, dumped from PromoStandards `ColorArray`. **Separate from color swatches** — this is raw per-variant color data, *not* the swatch mechanism (swatches = option-linked `custom-color-swatch` metaobjects; see "Color swatches"). | Array of objects |
| `gallery` | `list.file_reference` | **Gated.** Per-variant media gallery — featured image first, then color-matched extras | Array of Shopify File references |

### `gallery` (variant) — gated, may be absent

**The per-variant gallery metafield is only present on shops with `Shop.supports_variant_gallery == true`.** That property is true when **both** of these hold:

1. `psrestful_plan` is one of `TRIAL`, `BUSINESS`, `ENTERPRISE` (the paid tiers that include the multi-image-per-variant feature)
2. `variant_grouping_strategy != 'by_color'` (split-by-color shops have one product per color, so each product's native media gallery already covers it — a per-variant gallery would be redundant)

When the gate is closed, the metafield is **not written at all** — `variant.metafields.psrestful.gallery` will be empty/nil. Themes must check `.value.size > 0` before relying on it and fall back to `product.media` + `variant.featured_media`.

**Shape:** `list.file_reference` — a Liquid array of Shopify File objects. Each entry is typically a `MediaImage` (use `.image` to get the underlying Image object); occasionally a generic file (use `.url`). Order is meaningful:
- Index 0 = the variant's featured image (same URL as `variant.featured_media`)
- Index 1+ = additional color-matched media, deduplicated against the featured

**Color matching** is supplier-specific (PCNA / SanMar / S&S / etc.) via `MediaAssociationStrategyFactory` in the PromoSync app. Themes don't need to *invoke* the matching logic — they just consume the resulting ordered list — but knowing which strategy ran helps interpret edge cases (sparse galleries, surprising image counts). See "Media association strategies" below.

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

See `integration-patterns.md` → "Variant gallery (gated)" for the full pattern including fallback and JS variant-switching.

### Media association strategies (how the gallery is built)

PromoStandards suppliers populate `MediaContent` records inconsistently — some tag every asset with a color, some leave color blank but always set `partId`, some send only product-level media. To produce the per-variant `psrestful.gallery` list, PromoSync runs a **supplier-specific strategy** that decides which `MediaContent` rows belong to which variant.

#### Selection

PromoSync picks the strategy from a fixed per-supplier registry. Today:

| `supplier_code` | Strategy class | Why |
|---|---|---|
| `SanMar` | `DefaultMediaAssociationStrategy` | SanMar populates `MediaContent.color`, so color matching works as a fallback to partId |
| `SS` (S&S Activewear) | `DefaultMediaAssociationStrategy` | Same — color is reliably populated |
| `HIT` (HIT Promotional Products) | `PartIdOnlyMediaAssociationStrategy` | HIT leaves color blank; partId is the only reliable key, so no color fallback to avoid mis-association |
| Any other supplier (incl. PCNA, Carolina Made, Digital on Demand, unknown / `None`) | `DefaultMediaAssociationStrategy` (fallback) | Safe default — tries partId first, then color, then a fallback color field |

The registry is fixed per supplier; there is no per-shop override today.

#### What each strategy does

**`DefaultMediaAssociationStrategy`** — cascade per part:
1. Filter `MediaContent` where `partId == part.partId`. If non-empty, use it.
2. Else, read the part's **primary color** (`Shop.color_field`) and filter `MediaContent` by that color. If non-empty, use it.
3. Else, read the part's **fallback color** (`Shop.fallback_color_field`) and filter by that.
4. Dedupe URLs while preserving order.

This means a SanMar variant typically gets the union of (media tagged with its partId) OR (failing that, media tagged with its color). Cross-variant duplication is possible if multiple variants share a color and the supplier didn't tag partIds.

**`PartIdOnlyMediaAssociationStrategy`** — single-step:
1. Filter `MediaContent` where `partId == part.partId`. Dedupe. That's it.

No color fallback. If a HIT variant has no partId-tagged media, its gallery list is empty (the importer then writes only the featured image, or skips the metafield entirely if there's no featured either).

#### Implications for theme work

- **HIT galleries may be sparse.** Don't be surprised by a HIT variant with only 1–2 entries even on a paid plan — the strategy deliberately won't fall back to color.
- **SanMar / S&S galleries are usually rich** (5–10 per color is common) because the cascade pulls color-matched extras when partId-matched media is thin.
- **Unmapped suppliers run the default cascade.** Expect partId match first, color fallback after. If a supplier's color tagging is inconsistent across variants, expect uneven gallery sizes.
- **An empty gallery list isn't necessarily a bug.** It can mean the supplier didn't tag any media against this part or its color. The theme's job is to fall back to `product.media` gracefully (see the integration pattern).
- **Order in the gallery list reflects the order the strategy returned URLs**, with the featured image hoisted to index 0. Don't expect a curated sequence; if you need "primary, then alternates," index-0 is reliable but index-1+ is supplier-driven.

### Blank / sample metafields (`blank_minimum_quantity`, `blank_part_price_array`, `sample_price`) — gated trio

**These three metafields are only present on shops where `Shop.emit_blank_sample_metafields == true`.** That property is true when **both** of these hold:

1. `Shop.enable_blank_sample_sales == True` (admin-only flag — merchants don't toggle it themselves)
2. `Shop.default_configuration_type == 'Decorated'` (i.e. the shop is selling *decorated* products by default — the regular `minimum_quantity` and `part_price_array` hold the **decorated** values)

The point is to let a Decorated-default shop **also** surface blank pricing and single-unit sample pricing on the storefront, so customers can choose between **Decorated**, **Blank**, and **Sample** purchase modes on the same product.

**Why not on Blank-default shops?** On a Blank-default shop, the regular `minimum_quantity` and `part_price_array` already *are* the blank values — emitting `blank_minimum_quantity` and `blank_part_price_array` would be the same data twice. The gate prevents that duplication.

**Key invariants:**

- `blank_part_price_array` has the **exact same JSON shape** as `part_price_array` (`[{quantityMin, price-in-cents}, …]`, ascending by `quantityMin`, last match wins). Reuse the same matching logic — just swap the metafield path.
- `blank_minimum_quantity` and `blank_part_price_array` are written **independently** per product/variant. A product with no blank pricing data from the supplier won't get them even when the gate is open. Themes must `if metafield.value.size > 0` / `if metafield`.
- `sample_price` is **derived**, not configured. It's `min(blank_part_price_array, key=quantityMin).price` *only when* the lowest tier's `quantityMin == 1`. If the supplier's blank tiers start at qty 6, `sample_price` is omitted — selling a single sample isn't fulfillable.
- `sample_price` is **in cents**, like all PromoSync price fields. Use `money_without_currency` directly.
- The **decorated** values stay on `minimum_quantity` / `part_price_array` — there is no `decorated_part_price_array`. So a Decorated-default shop with the gate open has: `part_price_array` = decorated tiers, `blank_part_price_array` = blank tiers, `sample_price` = qty-1 blank shortcut.

**Liquid access:**

```liquid
{%- comment -%} Decorated (default) {%- endcomment -%}
{%- assign dec_min = product.metafields.psrestful.minimum_quantity -%}
{%- assign dec_tiers = variant.metafields.psrestful.part_price_array.value -%}

{%- comment -%} Blank — gated, may be nil {%- endcomment -%}
{%- assign blank_min = product.metafields.psrestful.blank_minimum_quantity -%}
{%- assign blank_tiers = variant.metafields.psrestful.blank_part_price_array.value -%}

{%- comment -%} Sample — gated AND requires qty-1 blank tier; may be nil even when blank_tiers exists {%- endcomment -%}
{%- assign sample_cents = variant.metafields.psrestful.sample_price -%}
```

See `integration-patterns.md` → "Blank / Decorated / Sample purchase flows" for the full add-to-cart, line-item-property, and pricing-display patterns.

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
- `price` is in **cents** (2500 = $25.00)
- Tiers are sorted by `quantityMin` ascending
- **Match by iterating all tiers and taking the last one where `cart_quantity >= quantityMin`**
- Use `{{ tier.price | money_without_currency }}` — do not divide by 100 first
- The Shopify Function `quantity-price-breaks` reads this metafield and applies actual discount at checkout

### `mm-google-shopping.*` (variant)

Only present when `Shop.enable_google_merchant=true`.

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

### `psrestful.*` (variant) — Hit a Double multi-location inventory

Only present when `Shop.is_hit_a_double=true`.

| Key | Type | Purpose |
|-----|------|---------|
| `ProductID_VariantID` | `single_line_text_field` | US Shopify ProductID/VariantID mapping |
| `variant_key` | `single_line_text_field` | DropShip variant/inventory key |
| `variant_sm_key` | `single_line_text_field` | SanMar variant/inventory key |
| `variant_ss_key` | `single_line_text_field` | S&S Activewear variant/inventory key |
| `variant_cm_key` | `single_line_text_field` | Carolina Made variant/inventory key |

---

## Color swatches (option-linked metaobjects)

PromoSync renders **native** Shopify color swatches by linking a product's **Color
option** to **metaobjects** — not by writing a value the theme parses ad hoc.
(Hand-verified 2026-06-09.)

### The two definitions (created once per shop)

`ColorSwatchService.ensure_definitions()` lazily creates these and caches their GIDs on the `Shop`:

1. **Metaobject definition** — type `custom-color-swatch`, name "Color Swatch",
   `access.storefront = PUBLIC_READ`. Fields:

   | Field key | Type | Holds |
   |---|---|---|
   | `label` | `single_line_text_field` | Color name, e.g. `"Red"` |
   | `hex_color` | `single_line_text_field` | Hex, e.g. `"#FF0000"` |

   GID cached on `Shop.color_metaobject_definition_id`.

2. **Product metafield definition** — namespace `psrestful`, key `color-swatches`,
   **type `list.metaobject_reference`**, ownerType `PRODUCT`, validated to the
   metaobject definition above, `access.storefront = PUBLIC_READ`. GID cached on
   `Shop.color_metafield_definition_id`.

### Per-color metaobjects

For each color **that has a hex**, `metaobjectUpsert` creates/updates one metaobject:
- type `custom-color-swatch`, handle = `slugify(color_name)`
- fields `label` = color name, `hex_color` = hex
- color_name→GID cached 24h at `color_swatch_mo:{shop_id}:{slug}`

Colors **without** a hex are skipped — no metaobject, no swatch.

### Where the hex comes from

`ExtraService.get_color_hex_map(supplier_code, product_id)` calls psrestful-api
`GET extra/v2/colors/{supplier_code}/product/{product_id}`:

```json
{ "Red": { "hex": "#FF0000", "colorGroup": "Red", "source": "normalized" } }
```

### The actual link (what makes swatches render)

The Color option is built **option-linked** (`_build_color_option`). The option
payload (productSet / productOptionUpdate) looks like:

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

Each **option value** carries `linkedMetafieldValue` = the metaobject GID. With no
metaobject map the option degrades to plain `{ "name": "Red" }` (no swatch).

### `color_field` preference

`Shop.color_field` ∈ { `colorName` (default), `standardColorName` } selects which
PromoStandards color field is the primary color identifier. `get_variant_color`
falls back to the other field, then to the part's primary color.

### Gating

`Shop.supports_color_swatches == (enable_color_swatches OR _has_plan_gated_access)`.
Off by default; on for plan-gated shops or when a merchant enables it. When off,
the Color option is unlinked and no swatch metaobjects/metafield are written.

### Adding variants later

`variant_adder` enriches the option's linked map with shop-wide metaobjects
(`_resolve_shop_metaobjects`). If a **new** variant's color has no linked
metaobject, it **raises** — *"color X not linked to any metaobject … Merchant must
add a swatch for this color and retry."* A missing swatch **blocks** the variant
add; it does not silently skip.

### Reading swatches in a theme (Liquid)

`color-swatches` is `list.metaobject_reference`, so `.value` is an **array** of
`custom-color-swatch` metaobjects, each exposing `.label` and `.hex_color`.

> ⚠️ **Hyphenated key gotcha.** `product.metafields.psrestful.color-swatches` parses
> the hyphen as subtraction. You **must** use bracket access:
> `product.metafields.psrestful["color-swatches"].value`.

```liquid
{%- assign swatches = product.metafields.psrestful["color-swatches"].value -%}
{%- for swatch in swatches -%}
  {{ swatch.label }} → {{ swatch.hex_color }}
{%- endfor -%}
```

Length via `…["color-swatches"].value.count` (reference lists use `count`, not `size`).
Because the metaobject is a **custom** type (not Shopify's standard color/pattern
metaobject), `option_value.swatch` is **not** auto-populated — a custom theme should
read this list and match `swatch.label` to the option value name. See
`integration-patterns.md` → "Color swatches" for the full render snippet.

---

## Configurable products (multi-part-group: kits, sets, bundles)

**Gated by `Shop.enable_configurable_products` (admin flag, default off).** A small but growing class of supplier products are *configurable*: PPC describes them as multiple **part groups** (a plush bear + an apparel item, a tumbler + a color-matched straw, a 6-piece new-hire kit). When the flag is on **and** the PPC response has more than one part group, PromoSync builds the **real variant matrix** from the part-group structure instead of flattening every part into one option. Regular single-group products are completely unaffected (byte-for-byte identical to before).

To the theme, a configurable product is **just a normal multi-option Shopify product** — read its metafields exactly as you would any other. The differences worth knowing:

- **Option names can be generic.** Each part-group "axis" becomes a Shopify option. When the supplier's group label is the generic `"Part Group"`, the option name falls back to `Option 1`, `Option 2`, …; otherwise it's the supplier label (e.g. `SUNGLASSES`). **Don't assume an option is named `Color`.**
- **Option values may be disambiguated.** When one axis mixes sub-types (the apparel axis spans SHIRT / HOODIE / BANDANA, each with a "NAVY BLUE"), values are prefixed to stay unique: `SHIRT - NAVY BLUE`, `HOODIE - NAVY BLUE`. Render them as-is.
- **SKU is a joined bill-of-materials.** A variant's SKU concatenates every component partId with `-` — e.g. `1200BRN-1210BLK` (bear + shirt) or `5306RBLK-5306RBLKSTRAW` (tumbler + straw). **Never parse the SKU to recover a partId** — use the `psrestful.part_id` metafield.
- **`psrestful.part_id` is the *priced* component, not the whole SKU.** Each configurable variant's `part_id` is the priced part of the combination (the apparel for the bear, the body for the tumbler, the anchor for a kit). Per-part lookups (inventory, decoration enrichment) key off this, not the joined SKU.
- **`part_price_array` works exactly as normal** — the variant's tier array is the priced component's tiers (cents, ascending). Tier-pricing display, the volume Shopify Function, and `minimum_quantity` need no special handling.
- **`location_decorations` is the UNION across the combination's parts.** A bear+shirt variant exposes both the bear's and the shirt's decoration locations; a tumbler exposes the body's locations (the straw has none). Decoration pricing reads it the same way.
- **Variant images** are resolved per variant from the best-matching component (exact `partId` → shared color-code → product group image), so `variant.featured_media` is set normally and the `gallery` gating rules are unchanged.

**Bottom line for theme work:** treat configurable products like any multi-option product. The only practical gotchas are (a) option names may be `Option N`, (b) option values may carry a `TYPE - COLOR` prefix, and (c) the SKU is a joined BOM — always prefer `psrestful.part_id` over the SKU.

---

## Shop-level configuration (feature flags)

These Django model fields on `Shop` control which metafields get written. If a metafield is missing on a store, check the flag.

| Field | Effect |
|-------|--------|
| `enable_google_merchant` | Adds `mm-google-shopping.*` on products and variants |
| `enable_color_swatches` | Turns on native color swatches: creates the `custom-color-swatch` metaobject + `psrestful.color-swatches` (`list.metaobject_reference`) definitions and links the Color option's values to per-color metaobjects. See "Color swatches". |
| `supports_color_swatches` (property) | Computed: `enable_color_swatches OR _has_plan_gated_access`. The actual gate the importer checks before writing swatch metaobjects/links. |
| `color_field` | `'colorName'` (default) or `'standardColorName'` — which PromoStandards color field is the primary color identifier (with fallback to the other). |
| `add_default_tier_pricing` | Overrides supplier tier pricing with distributor rules in `psrestful.part_price_array` |
| `add_default_location_decorations` | Augments `psrestful.location_decorations` with default locations/methods |
| `enable_decoration_tags` | Admin-only (default off). When on, tags imported products with their decoration methods. Part of the v3 decoration-products surface (migration 0076). |
| `extra_id` (product metafield, not a flag) | PromoSync's internal product id. **Required for the decoration enricher** — without it, no `*VariantId` fields are written to `location_decorations` and decorations stay unpriced. See `decoration-pricing.md`. |
| `supports_variant_gallery` (property) | Writes `psrestful.gallery` on each variant. **True only when `psrestful_plan ∈ {TRIAL, BUSINESS, ENTERPRISE}` AND `variant_grouping_strategy != 'by_color'`.** When false, the metafield is skipped entirely — themes must fall back to `product.media`. |
| `enable_blank_sample_sales` | Admin-only flag. Combined with `default_configuration_type='Decorated'`, this opens the gate for `blank_minimum_quantity`, `blank_part_price_array`, and `sample_price` metafields so the storefront can sell **Decorated**, **Blank**, and **Sample** variants of the same product. |
| `default_configuration_type` | `'Decorated'` or `'Blank'` — drives what `minimum_quantity` and `part_price_array` mean. Decorated-default shops with the blank/sample gate open also get `blank_*` metafields; Blank-default shops never do (their regular metafields already are the blank values). |
| `emit_blank_sample_metafields` (property) | Computed: `enable_blank_sample_sales AND default_configuration_type == 'Decorated'`. The single check the importer uses to decide whether to write the blank/sample trio. |
| `import_minimum_quantity_one` | Forces `psrestful.minimum_quantity = 1` regardless of supplier value |
| `enable_configurable_products` | Admin flag (default off). For PPC products with **>1 part group** (kits/sets/bundles — plush + apparel, tumbler + color-matched straw), builds the real variant matrix from the part-group structure instead of flattening parts into one option. Single-group products are unaffected. See "Configurable products". |
| `is_hit_a_double` | Adds `my_fields.*` + `psrestful.variant_*_key` multi-location fields |
| `enable_volume_pricing` | Activates the `quantity-price-breaks` Shopify Function |
| `color_metafield_definition_id` | Cached GID of color-swatches metafield definition |
| `color_metaobject_definition_id` | Cached GID of color-swatch metaobject definition |

---

## Write paths (who writes each namespace)

| Writer | Namespace(s) | Trigger |
|--------|--------------|---------|
| Product importer | all | Product import from PromoStandards |
| Update Metafields job | all | Manual update job / admin action |
| Color swatch service | `custom-color-swatch` metaobjects + `psrestful.color-swatches` definition | Creates the metaobject + metafield definitions; upserts per-color metaobjects (`label`/`hex_color`) |
| Product option builder (import and product update) | Color option `linkedMetafield` → `psrestful.color-swatches` values | Links each Color option value to its metaobject GID (`linkedMetafieldValue`) via productSet/productOptionUpdate |
| Decorations service | `psrestful.location_decorations`; canonical-product `method_slug` / `charge_kind` / `default_variant_id`; variant `pricing_hash` / `suppliers` / `part_price_array` | Decoration import / default application + "Insert decorations" canonical-product upsert |
| Decoration enricher | `psrestful.location_decorations` (`*VariantId` fields) | Decoration enricher — stamps canonical-variant GIDs onto each decoration entry (runs after "Insert decorations") |
| Product/variant builder | all | Metafield generation during product/variant create |
| Google Merchant generator | `mm-google-shopping` | Google Merchant field generation |

## Read paths (who reads each namespace)

| Reader | Keys |
|--------|------|
| Theme block `tier-pricing.liquid` | `psrestful.part_price_array` |
| Theme block `location-decorations.liquid` | `psrestful.location_decorations` |
| Theme block `min-quantity-notice.liquid` | `psrestful.minimum_quantity` |
| Theme block `promo-pricing-engine.liquid` (embed) | `psrestful.minimum_quantity`, `psrestful.part_price_array` |
| Theme block `volume-pricing-note.liquid` | `psrestful.part_price_array` |
| Shopify Function `quantity-price-breaks` | `psrestful.part_price_array` |
| Update Metafields job (variant matching) | `psrestful.part_id`, `psrestful.extra_id` |
| Importer (dedupe) | `psrestful.product_id`, `psrestful.supplier_code` |
| Theme gallery integrations (custom) | `psrestful.gallery` (variant — gated; falls back to `product.media` when absent) |
| Theme product-specs / features tab (custom) | `psrestful.marketing_points` (product — raw supplier copy, filter with an allow-list of `pointType`s) |
| Native Shopify variant picker / theme swatch UI | Color option's `linkedMetafield` → `psrestful.color-swatches` metaobjects (`label`, `hex_color`). Custom themes read `product.metafields.psrestful["color-swatches"].value` directly. |
| Theme Blank/Sample purchase UI (custom) | `psrestful.blank_minimum_quantity`, `psrestful.blank_part_price_array`, `psrestful.sample_price` (gated trio — only on Decorated-default shops with `enable_blank_sample_sales=true`) |
| Theme decoration-pricing UI (custom — v3) | enriched `psrestful.location_decorations` `*VariantId` fields + canonical-product `psrestful.default_variant_id` / variant `psrestful.part_price_array`. Not done by the shipped block — see `decoration-pricing.md`. |

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

The decoration enricher later adds `*VariantId` fields (and `locationId` / `decorationsIncluded` / etc.) to
this same metafield — see the enriched shape above and `decoration-pricing.md`.

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
