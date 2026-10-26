# PromoSync Decoration Pricing (v3 enriched `location_decorations`)

How to turn PromoSync's decoration data into **real prices** on the storefront: setup fees, run charges,
extra-color and extra-location surcharges (each split into a one-time *setup* and a per-unit *run*), and
PMS-match upcharges — then add the right cart lines.

This is the authoritative skill reference for the decoration-pricing surface PromoSync shipped in its v3
"Insert decorations" / enrichment work. Read it whenever a theme needs to **price** a decorated product,
not just list the available locations/methods.

> **Reality check — read this first.** The shipped `location-decorations` theme app block is **v2-era**:
> it only renders location/method *names* and writes `properties[Location]` / `properties[Decoration]` on
> add-to-cart. It does **not** read any `*VariantId` field, compute pricing, or add the decoration cart
> lines. **The v3 pricing model below is consumable only via custom theme code today.** Don't tell a
> merchant "just enable the block and it'll price decorations" — it won't.

---

## 1. The two-part architecture

PromoSync's "Insert decorations" admin action builds the pricing surface in two halves:

1. **Canonical decoration products** — a small, shop-wide catalog of Shopify products, one product per
   **(imprint method × charge kind)**. These hold the actual prices.
2. **Enriched `location_decorations`** — the existing product-level metafield on each *physical* product
   (apparel, drinkware, bags…), where PromoSync's **enricher** stamps `*VariantId` GIDs onto each
   decoration entry, pointing at the right variant of the right canonical product.

The theme reads the physical product's enriched `location_decorations`, follows each `*VariantId` to a
canonical variant, reads its price (and tier ladder, for run charges), computes the totals, and adds one
cart line per charge that applies.

```
Physical product "ARIEL WSA-CM15"
└─ psrestful.supplier_code      = "ARIEL"
└─ psrestful.extra_id           = 7587          ← required for enrichment to run
└─ psrestful.location_decorations = [ {location, decorations:[ {decorationName, ...VariantId fields} ]} ]
                                              │
              each *VariantId GID points at ──┘
                                              ▼
Canonical product "Screen Print Run Charge"  (handle: screen-print-run-charge)
└─ psrestful.method_slug        = "screen-print"
└─ psrestful.charge_kind        = "run_charge"
└─ psrestful.default_variant_id = gid://…/Variant/…   ← storefront fallback
   └─ variant V (one per pricing class)
      ├─ psrestful.pricing_hash     = "a1b2c3d4"
      ├─ psrestful.suppliers        = ["ARIEL","HIT"]
      └─ psrestful.part_price_array = [{quantityMin,price-in-cents}, …]   ← run charges only
```

**Supplier scoping.** At "Insert decorations" time the merchant multi-selects which suppliers their store
sources from, so the catalog stays tight (≈30–60 canonical products for a 2-supplier shop, not the 200+ a
full consolidation would make). The theme never needs to know about scoping — it just sees fewer products
and follows GIDs.

---

## 2. The seven charge kinds

For each canonical imprint method PromoSync upserts **up to seven** canonical products — one per charge
kind. A kind's product exists only when *some* selected supplier publishes that charge for that method, so
many methods have fewer than seven (Dome may have only a Run Charge; Decal only a Setup).

| `charge_kind` (code value) | Handle suffix             | `product_type`         | `*VariantId` field on the decoration entry | What it charges |
|----------------------------|---------------------------|------------------------|--------------------------------------------|-----------------|
| `setup_fee`                | `-setup-fee`              | `Setup Fee`            | `setupFeeVariantId`                        | One-time per order (e.g. burn a screen) |
| `run_charge`               | `-run-charge`             | `Run Charge`           | `runChargeVariantId`                       | Per unit — **tier-priced** via `part_price_array` |
| `extra_color_setup`        | `-extra-color-setup`      | `Extra Color Setup`    | `extraColorSetupVariantId`                 | One-time, per extra imprint color |
| `extra_color_run`          | `-extra-color-run`        | `Extra Color Run`      | `extraColorRunVariantId`                   | Per unit, per extra imprint color |
| `extra_location_setup`     | `-extra-location-setup`   | `Extra Location Setup` | `extraLocationSetupVariantId`              | One-time, per extra imprint location |
| `extra_location_run`       | `-extra-location-run`     | `Extra Location Run`   | `extraLocationRunVariantId`                | Per unit, per extra imprint location |
| `pms_match`                | `-pms-match`              | `PMS Match`            | `pmsMatchVariantId`                        | Per Pantone-matched color, **per order** (flat) |

**Why extra-color / extra-location are split into setup + run.** Some suppliers (notably HIT) bill *both*
a one-time setup AND a per-unit run for the same extra color or location, and **both apply at the same
time**. Treat the setup/run pair independently — apply whichever `*VariantId`s are present.

`KIND_HANDLE_SUFFIX` (for fallback handle derivation, see §4):

```js
const KIND_HANDLE_SUFFIX = {
  setupFeeVariantId:           '-setup-fee',
  runChargeVariantId:          '-run-charge',
  extraColorSetupVariantId:    '-extra-color-setup',
  extraColorRunVariantId:      '-extra-color-run',
  extraLocationSetupVariantId: '-extra-location-setup',
  extraLocationRunVariantId:   '-extra-location-run',
  pmsMatchVariantId:           '-pms-match',
};
```

Canonical-product vendor is always `PromoSync`; tags are `[<kind>, promosync_decoration]`.
Variant SKU is deterministic: `{method_slug}-{charge_kind}-{pricing_hash}` (e.g.
`screen-print-run_charge-a1b2c3d4`).

---

## 3. The enriched `location_decorations` metafield

`product.metafields.psrestful.location_decorations` (`json`). Same metafield as before — the enricher
**adds** the `*VariantId` fields to it; it does not create a new metafield.

### 3.1 Full enriched example (Ariel WSA-CM15)

```json
[
  {
    "locationId": 39263,
    "locationName": "Front ",
    "decorationsIncluded": 1,
    "defaultLocation": true,
    "minDecoration": 0,
    "maxDecoration": 1,
    "locationRank": 1,
    "decorations": [
      {
        "decorationId": 58350,
        "decorationName": "Heat Transfer",
        "maxImprintColors": 99,
        "default": false,
        "priceIncludes": false,
        "setupFeeVariantId": "gid://shopify/ProductVariant/66769262051373",
        "runChargeVariantId": "gid://shopify/ProductVariant/66769269784621"
      },
      {
        "decorationId": 58355,
        "decorationName": "Screen Print",
        "maxImprintColors": 6,
        "default": true,
        "priceIncludes": true,
        "setupFeeVariantId": "gid://shopify/ProductVariant/66769244782637",
        "runChargeVariantId": "gid://shopify/ProductVariant/66769247698989",
        "extraColorSetupVariantId": "gid://shopify/ProductVariant/66769250000111",
        "extraColorRunVariantId": "gid://shopify/ProductVariant/66769250000222",
        "extraLocationSetupVariantId": "gid://shopify/ProductVariant/66769252220973",
        "extraLocationRunVariantId": "gid://shopify/ProductVariant/66769252220988",
        "pmsMatchVariantId": "gid://shopify/ProductVariant/66769252450349"
      }
    ]
  }
]
```

### 3.2 Field reference

**Location entry** (one per imprint location):

| Field                 | Meaning |
|-----------------------|---------|
| `locationId`          | Supplier's location id — internal, don't surface to customers. |
| `locationName`        | Human label ("Front", "Left Chest", "Back Pocket"). May have trailing whitespace — trim it. |
| `decorationsIncluded` | How many decorations come at no extra-location charge for this location. Feeds the extra-location math. |
| `defaultLocation`     | If `true`, default-select this location in the UI. |
| `minDecoration`       | Minimum decorations the customer must pick here. |
| `maxDecoration`       | Maximum decorations allowed here (often `1`). `0` → location exists but decoration isn't allowed for this product — skip rendering. |
| `locationRank`        | Display order (lower = earlier). |
| `decorations[]`       | Decoration options available at this location. |

**Decoration entry** (the unit the theme renders as a "decoration option"):

| Field                          | Meaning |
|--------------------------------|---------|
| `decorationId`                 | Supplier's decoration id — internal, don't surface. |
| `decorationName`               | **Canonical method name** ("Heat Transfer", "Screen Print", "Embroidery"). Already resolved server-side via the alias table — use as-is. |
| `maxImprintColors`             | Max imprint colors the supplier supports here. Drives the color-count UI ceiling. |
| `default`                      | Pre-select this decoration when the location opens. |
| `priceIncludes`                | If `true`, the base product price already includes this decoration — show it but **don't** add setup/run lines. |
| `setupFeeVariantId`            | GID of the Setup Fee variant. Absent if the supplier publishes no setup fee for this method. |
| `runChargeVariantId`          | GID of the Run Charge variant. Carries `psrestful.part_price_array` (tier ladder). Absent if no per-item run charge. |
| `extraColorSetupVariantId`     | One-time charge per additional imprint color. Absent if none. |
| `extraColorRunVariantId`       | Per-unit charge per additional imprint color. Absent if none. |
| `extraLocationSetupVariantId`  | One-time charge per additional imprint location. Absent if none. |
| `extraLocationRunVariantId`    | Per-unit charge per additional imprint location. Absent if none. |
| `pmsMatchVariantId`            | Per Pantone-matched color, per order. Absent if none. |

**Key rule:** an **absent `*VariantId` means "this charge doesn't apply"** — *not* "missing data". If
`extraColorRunVariantId` is absent, the supplier has no per-unit extra-color charge for this method; don't
surface an extra-color upsell for it.

### 3.3 Legacy fields the enricher strips (don't read these)

The enricher removes these on **every** run, so they should never appear in fresh data:

- `setupFeeProductId`, `runChargeProductId`, `extraColorProductId`, `extraLocationProductId`,
  `pmsMatchProductId` — **legacy v2** product-pointer fields (v3 points at variants, not products).
- `extraColorVariantId`, `extraLocationVariantId` — **pre-split v3** fields, replaced by the
  `*Setup`/`*Run` pairs. Use the new pairs.

---

## 4. Variant resolution (per decoration, per kind)

Prefer the per-product `*VariantId`. When it's absent — common for SanMar / S&S Activewear blanks, where
PSRESTful publishes no per-product decoration pricing — fall back to the canonical product's
`default_variant_id` metafield. If that's also absent, the charge doesn't apply.

```
if dec["{kind}VariantId"] is set:
    use dec["{kind}VariantId"]                      # per-product pricing — preferred
else:
    handle  = slugify(dec.decorationName) + KIND_HANDLE_SUFFIX[kind]
    variant = productByHandle(handle).metafield(psrestful.default_variant_id).reference
    if variant is not null:
        use variant                                 # merchant-defined / auto-picked default
    else:
        do not charge this kind                     # supplier doesn't offer it & no default
```

`default_variant_id` is auto-picked by PromoSync as the pricing class with the highest product count
across the chosen suppliers; the merchant can pin a different class in the PromoSync admin. It rarely
changes, so **cache the resolved default per `method_slug`+kind for the request's lifetime.**

Storefront API — resolve a default and read its price + ladder in one round trip:

```graphql
query CanonicalDefault($handle: String!) {
  productByHandle(handle: $handle) {
    metafield(namespace: "psrestful", key: "default_variant_id") {
      reference {
        ... on ProductVariant {
          id
          price { amount currencyCode }
          metafield(namespace: "psrestful", key: "part_price_array") { value }
        }
      }
    }
  }
}
```

The theme does **not** need to read `suppliers` or `pricing_hash` — the enricher already used the
product's per-product `pricingHash` to choose the exact variant. Those metafields exist for PromoSync's
own matching and for humans scanning the variant grid.

---

## 5. Pricing math

Read each resolved variant's `price` (and `part_price_array` for run charges). Inputs from the UI:
`qty` (units of the blank), `chosen_colors`, `chosen_locations`, `pms_colors`. Two "included" baselines:

- `included_colors` — colors included in the base run before extra-color kicks in. Comes from the
  supplier-level configuration; **default to 1** unless you have a better value.
- `decorationsIncluded` — from the **location entry**; how many decorations are included before
  extra-location kicks in.

```
extras_colors    = max(0, chosen_colors    − included_colors)
extras_locations = max(0, chosen_locations − decorationsIncluded)
```

### 5.1 Setup fee, extra-*-setup, PMS match (scalar `price`)

The variant's `price` is the headline list price (dollars from the Storefront API; cents if read from the
canonical `part_price_array`). Apply only the components whose `*VariantId` is present.

```
setup_fee_total        = setup_fee_price                                   (one-time, ×1)
extra_color_setup_total    = extra_color_setup_price    × extras_colors    (one-time)
extra_location_setup_total = extra_location_setup_price × extras_locations (one-time)
pms_match_total        = pms_match_price × pms_colors                      (flat — NOT × qty)
```

### 5.2 Run charge (tier-priced via `part_price_array`)

The run-charge variant's `price` is just the qty-1 / smallest-tier price. The real schedule is on
`psrestful.part_price_array` — JSON, **integer cents**, ascending by `quantityMin`, **last tier with
`quantityMin <= qty` wins**. PromoSync collapses adjacent equal-price tiers, so every entry is a real
price step.

```json
[ { "quantityMin": 1, "price": 85 }, { "quantityMin": 144, "price": 70 }, { "quantityMin": 288, "price": 55 } ]
```

```js
function tierPriceCents(ladder, qty) {
  let chosen = ladder[0];
  for (const tier of ladder) {
    if (tier.quantityMin <= qty) chosen = tier;
    else break;                       // already sorted ascending by quantityMin
  }
  return chosen.price;                // integer cents
}
```

```
run_charge_total       = tierPriceCents(run_ladder, qty)        × qty       (per unit)
extra_color_run_total      = extra_color_run_price    × extras_colors    × qty
extra_location_run_total   = extra_location_run_price × extras_locations × qty
```

> **Extra-color / extra-location can have BOTH a setup and a run component on the same product.** Add
> whichever of the two are present; never assume one excludes the other.

### 5.3 Decoration total

```
decoration_total =
    setup_fee_total
  + run_charge_total
  + extra_color_setup_total    + extra_color_run_total
  + extra_location_setup_total + extra_location_run_total
  + pms_match_total
```

If `priceIncludes == true` for the chosen decoration, **skip the setup/run terms** (the base price already
covers them). Extra-color / extra-location / PMS may still apply if their `*VariantId`s are present.

### 5.4 Worked example

144 units, **Screen Print** (`priceIncludes:false`), customer picks **3 colors** (`included_colors=1`)
across **2 locations** (`decorationsIncluded=1`), and wants **1 PMS match**. Resolved prices:
setup $50.00, run ladder `[{1,85},{144,70}]`, extra-color setup $12.00 + run $0.10, extra-location setup
$68.50 + run $0.05, PMS match $25.00.

```
extras_colors    = max(0, 3 − 1) = 2
extras_locations = max(0, 2 − 1) = 1

setup_fee            = $50.00                                   →  $50.00
run_charge           = tier(144) = $0.70 × 144                  →  $100.80
extra_color_setup    = $12.00 × 2                               →  $24.00
extra_color_run      = $0.10 × 2 × 144                          →  $28.80
extra_location_setup = $68.50 × 1                               →  $68.50
extra_location_run   = $0.05 × 1 × 144                          →  $7.20
pms_match            = $25.00 × 1   (flat)                      →  $25.00
                                                          decoration_total = $304.30
```

---

## 6. Adding decorations to the cart

Add one Shopify cart line per charge that applies, each at the correct quantity. Customer's blank `qty`
drives the per-unit lines.

| Line                  | Cart quantity                          |
|-----------------------|----------------------------------------|
| Physical product      | `qty`                                  |
| Setup fee             | `1`                                    |
| Run charge            | `qty`                                  |
| Extra color setup     | `extras_colors`                        |
| Extra color run       | `qty × extras_colors`                  |
| Extra location setup  | `extras_locations`                     |
| Extra location run    | `qty × extras_locations`               |
| PMS match             | `pms_colors`                           |

`/cart/add.js` takes **numeric** variant ids, not GIDs:

```js
const numericId = gid => Number(gid.split('/').pop());

await fetch('/cart/add.js', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    items: [
      { id: physicalVariantId, quantity: 144 },
      { id: numericId(dec.setupFeeVariantId),  quantity: 1 },
      { id: numericId(dec.runChargeVariantId), quantity: 144,
        properties: { _promosync_pricing_hash: 'a1b2c3d4' } },
      // …extra-color / extra-location / PMS lines, only when their *VariantId is present
    ]
  })
});
```

**Tier price enforcement at checkout.** For run charges, Shopify charges the variant's *headline* (qty-1)
price unless you override it. Two paths:

- **cart-transform Shopify Function** (recommended): read `part_price_array`, look up the tier for `qty`,
  override the line price.
- **Draft order checkout:** build the order with computed prices server-side.

A pure-Liquid theme without one of these shows the correct tier price to the customer but charges the
headline price — reconcile at order-management time, or set expectations with the merchant.

**Drop lines the customer didn't pick.** If a location isn't enabled, drop all its decoration lines. If
the customer wants no PMS match, drop the `pmsMatchVariantId` line. Don't send zero-quantity lines.

---

## 7. Edge cases & rules

- **`priceIncludes == true`** — show the decoration, skip its setup/run lines (extras/PMS may still apply).
- **`maxDecoration == 0`** — location exists but decoration isn't allowed for this product; skip it.
- **Absent `*VariantId`** — first fall back to the canonical product's `default_variant_id` (§4); if still
  absent, the charge truly doesn't apply. Never show a stale price.
- **Personalization decorations** — names ending `- Personalization` collapse to the same canonical method
  as the base ("Heat Transfer - Personalization" → "Heat Transfer"). When the supplier publishes distinct
  personalization pricing, the canonical product has a separate variant and the enricher picks it by hash.
  The theme just consumes whatever `*VariantId` it's handed.
- **Whole decoration unenriched** (every `*VariantId` absent on a method the catalog clearly covers) —
  almost always the physical product is **missing `psrestful.extra_id`**. The enricher needs `extra_id`
  to fetch per-product pricingHashes from PSRESTful; without it, it writes nothing rather than guess.
  Fix: backfill `extra_id` in PromoSync and re-run the enricher.
- **Unmatched method** — if PromoSync couldn't classify a decoration to a canonical method (rare; usually
  a brand-new PSRESTful method), every `*VariantId` is absent. Treat as "available but uncharged" or hide.
- **Idempotent re-runs** — the enricher is safe to run repeatedly; identical data is skipped silently.
  Theme code should be resilient to the metafield being byte-identical across renders.

---

## 8. Cross-checking

If a field name or value here disagrees with what you see in a live store, trust the live metafield and check https://promosync-docs.psrestful.com for the current shape.

See also: `metafields.md` (the canonical-product + enriched-shape catalog entries),
`integration-patterns.md` → "Decoration pricing (v3 enriched `location_decorations`)" for adapt-ready JS,
and `theme-blocks.md` (why the shipped block doesn't price decorations).
