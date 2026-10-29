# PromoSync Decoration Pricing (enriched `location_decorations`)

How to turn PromoSync's decoration data into real prices on the storefront: setup fees, run charges,
extra-color and extra-location surcharges (each split into a one-time *setup* and a per-unit *run*), and
PMS-match upcharges, then add the right cart lines.

Read it whenever a theme needs to **price** a decorated product, not just list the available locations and
methods.

> **Reality check, read this first.** The shipped `location-decorations` theme app block only renders
> location and method *names* and writes `properties[Location]` / `properties[Decoration]` on add-to-cart.
> It does **not** read any `*VariantId` field, compute pricing, or add decoration cart lines. **The pricing
> model below is consumable only through custom theme code today.** Don't tell a merchant "just enable the
> block and it will price decorations"; it won't.

---

## 1. The two-part architecture

PromoSync's **Insert Decorations** action builds the pricing surface in two halves:

1. **Canonical decoration products**: a small, shop-wide catalog of Shopify products, one product per
   **(imprint method × charge kind)**. These hold the actual prices.
2. **Enriched `location_decorations`**: the existing product-level metafield on each *physical* product
   (apparel, drinkware, bags), where the **Location Decorations Enrichment** job stamps `*VariantId` GIDs
   onto each decoration entry, pointing at the right variant of the right canonical product.

The theme reads the physical product's enriched `location_decorations`, follows each `*VariantId` to a
canonical variant, reads its price (and tier ladder, for run charges and setup tiers), computes the totals,
and adds one cart line per charge that applies.

```
Physical product "Classic Cotton Tee"
└─ psrestful.supplier_code      = "SUPPLIER"
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

**Supplier scoping.** When running Insert Decorations the merchant selects which suppliers their store
sources from, so the catalog stays tight (roughly 30 to 60 canonical products for a two-supplier shop
rather than hundreds). The theme never needs to know about scoping; it just sees fewer products and follows
GIDs.

---

## 2. The eight charge kinds

For each canonical imprint method PromoSync upserts **up to eight** canonical products, one per charge kind.
A kind's product exists only when some selected supplier publishes that charge for that method, so many
methods have fewer (a dome method may have only a run charge; a decal only a setup).

| `charge_kind` | Handle suffix | `product_type` | `*VariantId` field on the decoration entry | What it charges |
|---|---|---|---|---|
| `setup_fee` | `-setup-fee` | `Setup Fee` | `setupFeeVariantId` | One-time per order, flat (for example burning a screen) |
| `setup_tier` | `-setup-tier` | `Setup Tier` | `setupTierVariantId` | One-time per order, priced by decoration units (imprint colors or stitch count) through `psrestful.setup_tier_array`. Preferred over the flat setup fee when both are present |
| `run_charge` | `-run-charge` | `Run Charge` | `runChargeVariantId` | Per unit, **tier-priced** via `part_price_array` |
| `extra_color_setup` | `-extra-color-setup` | `Extra Color Setup` | `extraColorSetupVariantId` | One-time, per extra imprint color |
| `extra_color_run` | `-extra-color-run` | `Extra Color Run` | `extraColorRunVariantId` | Per unit, per extra imprint color |
| `extra_location_setup` | `-extra-location-setup` | `Extra Location Setup` | `extraLocationSetupVariantId` | One-time, per extra imprint location |
| `extra_location_run` | `-extra-location-run` | `Extra Location Run` | `extraLocationRunVariantId` | Per unit, per extra imprint location |
| `pms_match` | `-pms-match` | `PMS Match` | `pmsMatchVariantId` | Per Pantone-matched color, **per order** (flat) |

**Why extra-color and extra-location are split into setup plus run.** Some suppliers bill *both* a one-time
setup and a per-unit run for the same extra color or location, and both apply at the same time. Treat the
setup/run pair independently: apply whichever `*VariantId`s are present.

`KIND_HANDLE_SUFFIX` (for fallback handle derivation, see section 4):

```js
const KIND_HANDLE_SUFFIX = {
  setupFeeVariantId:           '-setup-fee',
  setupTierVariantId:          '-setup-tier',
  runChargeVariantId:          '-run-charge',
  extraColorSetupVariantId:    '-extra-color-setup',
  extraColorRunVariantId:      '-extra-color-run',
  extraLocationSetupVariantId: '-extra-location-setup',
  extraLocationRunVariantId:   '-extra-location-run',
  pmsMatchVariantId:           '-pms-match',
};
```

Canonical-product vendor is always `PromoSync`; tags are `[<kind>, promosync_decoration]`. Variant SKU is
deterministic: `{method_slug}-{charge_kind}-{pricing_hash}` (for example `screen-print-run_charge-a1b2c3d4`).

---

## 3. The enriched `location_decorations` metafield

`product.metafields.psrestful.location_decorations` (`json`). Same metafield as before; enrichment **adds**
the `*VariantId` fields to it and does not create a new metafield.

### 3.1 Full enriched example

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
        "setupFeeVariantId": "gid://shopify/ProductVariant/2001",
        "runChargeVariantId": "gid://shopify/ProductVariant/2002"
      },
      {
        "decorationId": 58355,
        "decorationName": "Screen Print",
        "maxImprintColors": 6,
        "default": true,
        "priceIncludes": true,
        "setupFeeVariantId": "gid://shopify/ProductVariant/1001",
        "setupTierVariantId": "gid://shopify/ProductVariant/1008",
        "runChargeVariantId": "gid://shopify/ProductVariant/1002",
        "extraColorSetupVariantId": "gid://shopify/ProductVariant/1003",
        "extraColorRunVariantId": "gid://shopify/ProductVariant/1004",
        "extraLocationSetupVariantId": "gid://shopify/ProductVariant/1005",
        "extraLocationRunVariantId": "gid://shopify/ProductVariant/1006",
        "pmsMatchVariantId": "gid://shopify/ProductVariant/1007"
      }
    ]
  }
]
```

### 3.2 Field reference

**Location entry** (one per imprint location):

| Field | Meaning |
|---|---|
| `locationId` | Supplier's location id; don't surface it to customers. |
| `locationName` | Human label ("Front", "Left Chest", "Back Pocket"). May have trailing whitespace; trim it. |
| `decorationsIncluded` | How many decorations come at no extra-location charge for this location. Feeds the extra-location math. |
| `defaultLocation` | If `true`, default-select this location in the UI. |
| `minDecoration` | Minimum decorations the customer must pick here. |
| `maxDecoration` | Maximum decorations allowed here (often `1`). `0` means the location exists but decoration isn't allowed for this product; skip rendering. |
| `locationRank` | Display order (lower = earlier). |
| `decorations[]` | Decoration options available at this location. |

**Decoration entry** (the unit the theme renders as a "decoration option"):

| Field | Meaning |
|---|---|
| `decorationId` | Supplier's decoration id; don't surface it. |
| `decorationName` | **Canonical method name** ("Heat Transfer", "Screen Print", "Embroidery"), already alias-resolved by PromoSync. Use as-is. |
| `maxImprintColors` | Max imprint colors the supplier supports here. Drives the color-count UI ceiling. |
| `default` | Pre-select this decoration when the location opens. |
| `priceIncludes` | If `true`, the base product price already includes this decoration; show it but **don't** add setup or run lines. |
| `setupFeeVariantId` | GID of the flat Setup Fee variant. Absent if the supplier publishes no setup fee for this method. |
| `setupTierVariantId` | GID of the Setup Tier variant. Carries `psrestful.setup_tier_array` (setup priced by imprint colors or stitches). Absent when the supplier prices setup as a flat fee only. |
| `runChargeVariantId` | GID of the Run Charge variant. Carries `psrestful.part_price_array` (tier ladder). Absent if no per-item run charge. |
| `extraColorSetupVariantId` | One-time charge per additional imprint color. Absent if none. |
| `extraColorRunVariantId` | Per-unit charge per additional imprint color. Absent if none. |
| `extraLocationSetupVariantId` | One-time charge per additional imprint location. Absent if none. |
| `extraLocationRunVariantId` | Per-unit charge per additional imprint location. Absent if none. |
| `pmsMatchVariantId` | Per Pantone-matched color, per order. Absent if none. |

**Key rule:** an **absent `*VariantId` means "this charge doesn't apply"**, not "missing data". If
`extraColorRunVariantId` is absent, the supplier has no per-unit extra-color charge for this method; don't
surface an extra-color upsell for it.

### 3.3 Older fields removed by enrichment (don't read these)

Enrichment removes these on every run, so they should never appear in fresh data:

- `setupFeeProductId`, `runChargeProductId`, `extraColorProductId`, `extraLocationProductId`,
  `pmsMatchProductId`: older product-pointer fields (enrichment points at variants, not products).
- `extraColorVariantId`, `extraLocationVariantId`: pre-split fields, replaced by the `*Setup` / `*Run`
  pairs. Use the new pairs.

---

## 4. Variant resolution (per decoration, per kind)

Prefer the per-product `*VariantId`. When it's absent (common for blank-goods suppliers whose feeds publish no
per-product decoration pricing) fall back to the canonical product's `default_variant_id` metafield. If that's
also absent, the charge doesn't apply.

```
if dec["{kind}VariantId"] is set:
    use dec["{kind}VariantId"]                      # per-product pricing, preferred
else:
    handle  = slugify(dec.decorationName) + KIND_HANDLE_SUFFIX[kind]
    variant = product(handle).metafield(psrestful.default_variant_id).reference
    if variant is not null:
        use variant                                 # merchant-defined or auto-picked default
    else:
        do not charge this kind                     # supplier doesn't offer it and no default
```

`default_variant_id` is auto-picked by PromoSync as the pricing class shared by the most products across the
chosen suppliers; the merchant can pin a different class in PromoSync. It rarely changes, so **cache the
resolved default per `method_slug` plus kind for the request's lifetime.**

Storefront API: resolve a default and read its price and ladders in one round trip:

```graphql
query CanonicalDefault($handle: String!) {
  product(handle: $handle) {
    metafield(namespace: "psrestful", key: "default_variant_id") {
      reference {
        ... on ProductVariant {
          id
          price { amount currencyCode }
          partPrices: metafield(namespace: "psrestful", key: "part_price_array") { value }
          setupTiers: metafield(namespace: "psrestful", key: "setup_tier_array") { value }
        }
      }
    }
  }
}
```

The theme does **not** need to read `suppliers` or `pricing_hash`; enrichment already used the product's
per-product pricing hash to choose the exact variant. Those metafields exist for PromoSync's own matching and
for humans scanning the variant grid.

---

## 5. Pricing math

Read each resolved variant's `price` (plus `part_price_array` for run charges and `setup_tier_array` for setup
tiers). Inputs from the UI: `qty` (units of the blank), `chosen_colors`, `chosen_locations`, `pms_colors`,
and for stitch-priced methods a stitch count. Two "included" baselines:

- `included_colors`: colors included in the base run before extra-color kicks in. Comes from the
  supplier-level configuration; **default to 1** unless you have a better value.
- `decorationsIncluded`: from the **location entry**; how many decorations are included before
  extra-location kicks in.

```
extras_colors    = max(0, chosen_colors    - included_colors)
extras_locations = max(0, chosen_locations - decorationsIncluded)
```

### 5.1 Setup fee, extra setups, PMS match (scalar `price`)

The variant's `price` is the headline price (a decimal amount from the Storefront API; cents if read from
`part_price_array`). Apply only the components whose `*VariantId` is present.

```
setup_fee_total            = setup_fee_price                                   (one-time, ×1)
extra_color_setup_total    = extra_color_setup_price    × extras_colors        (one-time)
extra_location_setup_total = extra_location_setup_price × extras_locations     (one-time)
pms_match_total            = pms_match_price × pms_colors                      (flat, NOT × qty)
```

### 5.2 Setup tier (priced by colors or stitches via `setup_tier_array`)

When `setupTierVariantId` is present, use its ladder **instead of** the flat setup fee. The variant's `price`
is the one-unit tier; the real schedule is on `psrestful.setup_tier_array`:

```json
{ "uom": "Colors", "tiers": [ { "minUnits": 1, "price": 5500 }, { "minUnits": 2, "price": 9500 } ] }
```

`uom` names the axis: `Colors` (units = `chosen_colors`) or `Stitches` (units = the stitch count the
customer or the artwork implies). Tiers are ascending by `minUnits`, `price` in integer cents, equal-price
tiers collapsed; **the last tier with `minUnits <= units` wins**. The example reads "$55 for one color, $95
for two or more".

```
setup_tier_total = tier(setup_tier_array, units).price     (one-time, ×1)
```

### 5.3 Run charge (tier-priced via `part_price_array`)

The run-charge variant's `price` is just the qty-1 (smallest-tier) price. The real schedule is on
`psrestful.part_price_array`: JSON, **integer cents**, ascending by `quantityMin`, **last tier with
`quantityMin <= qty` wins**. PromoSync collapses adjacent equal-price tiers, so every entry is a real price
step.

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
run_charge_total         = tierPriceCents(run_ladder, qty)        × qty       (per unit)
extra_color_run_total    = extra_color_run_price    × extras_colors    × qty
extra_location_run_total = extra_location_run_price × extras_locations × qty
```

`qty` here follows the store's volume pricing mode: the line's own quantity in **Per variant** mode, or the
sum of every cart line of the same product in **Combined by product** mode. See `integration-patterns.md`,
"Match the store's volume pricing mode".

> **Extra-color and extra-location can have BOTH a setup and a run component on the same product.** Add
> whichever of the two are present; never assume one excludes the other.

### 5.4 Decoration total

```
decoration_total =
    (setup_tier_total if setupTierVariantId else setup_fee_total)
  + run_charge_total
  + extra_color_setup_total    + extra_color_run_total
  + extra_location_setup_total + extra_location_run_total
  + pms_match_total
```

If `priceIncludes == true` for the chosen decoration, **skip the setup and run terms** (the base price already
covers them). Extra-color, extra-location and PMS may still apply if their `*VariantId`s are present.

### 5.5 Worked example

144 units, **Screen Print** (`priceIncludes:false`, flat setup fee, no setup tier), customer picks **3 colors**
(`included_colors=1`) across **2 locations** (`decorationsIncluded=1`), and wants **1 PMS match**. Resolved
prices: setup $50.00, run ladder `[{1,85},{144,70}]`, extra-color setup $12.00 plus run $0.10,
extra-location setup $68.50 plus run $0.05, PMS match $25.00.

```
extras_colors    = max(0, 3 - 1) = 2
extras_locations = max(0, 2 - 1) = 1

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

Add one Shopify cart line per charge that applies. The customer's `qty` drives the per-unit lines, and
every charge line carries properties that tie it to the product line it belongs to.

| Line | Cart quantity | Notes |
|---|---|---|
| Physical product | `qty` | Carries `properties[Location]`, `properties[Decoration]` and the rest of the selection |
| Setup fee **or** setup tier | `1` | Setup tier when `setupTierVariantId` is present, else the flat setup fee |
| Run charge | `qty` | Same quantity as the product line, always |
| Extra color setup | `extras_colors` | One line |
| Extra color run | `qty`, **one line per extra color** | `_promosync_unit` = 2, 3, … keeps the lines separate |
| Extra location setup | `extras_locations` | One line |
| Extra location run | `qty`, **one line per extra location** | `_promosync_unit` = 2, 3, … |
| PMS match | `pms_colors` | One line |

**Line properties on every charge line.** Shopify has no native parent-child link between cart lines, so the
theme has to carry one:

- `_promosync_parent`: the physical variant's numeric id
- `_promosync_charge`: the charge kind (`setup_fee`, `run_charge`, `extra_color_run`, …)
- `_promosync_unit`: for per-extra lines, the index of the extra color or location, which keeps same-variant
  lines from merging
- `_promosync_pricing_hash` (optional): the run-charge variant's `pricing_hash`, for order processing

Properties that start with an underscore are hidden in Dawn-family cart templates and in checkout, but they
are still on the order.

`/cart/add.js` takes **numeric** variant ids, not GIDs:

```js
const numericId = gid => Number(gid.split('/').pop());

await fetch('/cart/add.js', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    items: [
      { id: physicalVariantId, quantity: 144,
        properties: { Location: 'Front', Decoration: 'Screen Print', Colors: '3' } },
      { id: numericId(dec.setupFeeVariantId),  quantity: 1,
        properties: { _promosync_parent: physicalVariantId, _promosync_charge: 'setup_fee' } },
      { id: numericId(dec.runChargeVariantId), quantity: 144,
        properties: { _promosync_parent: physicalVariantId, _promosync_charge: 'run_charge',
                      _promosync_pricing_hash: 'a1b2c3d4' } },
      { id: numericId(dec.extraColorRunVariantId), quantity: 144,
        properties: { _promosync_parent: physicalVariantId, _promosync_charge: 'extra_color_run', _promosync_unit: 2 } },
      { id: numericId(dec.extraColorRunVariantId), quantity: 144,
        properties: { _promosync_parent: physicalVariantId, _promosync_charge: 'extra_color_run', _promosync_unit: 3 } },
      // …extra-color setup, extra-location, PMS lines, only when their *VariantId is present
    ]
  })
});
```

**What checkout charges.** Shopify charges each line at its variant's price unless an app discounts it:

- **Run charge:** the run-charge variant carries `psrestful.part_price_array`, and PromoSync's **Quantity
  Price Breaks** Function reads that metafield on every cart line. So when the merchant has **Enable Volume
  Pricing** on (PromoSync > Shop Settings > Volume Pricing), checkout discounts the run-charge line to the
  tier for its quantity, on any Shopify plan. No cart transform, custom Function or Shopify Plus plan is
  needed. With Volume Pricing off, checkout charges the headline (qty-1) price and the merchant reconciles at
  order time. Ask which case the store is in and say so to the merchant.
- **Setup tier:** the variant's price is the one-unit tier, and a discount can only lower a price, so
  checkout cannot charge the two-color or three-color tier. Show the right amount in the cart and tell the
  merchant that multi-color setups need a draft order or an adjustment at order processing.
- **Setup fee, extra setups, extra runs, PMS match:** flat per-unit prices, charged correctly by quantity.

In **Combined by product** mode the Function sums quantities per product id, which for charge lines means per
canonical charge product: two decorated products that share a run-charge variant get the tier for their
combined quantity. Keeping run-charge lines at the product quantity is what makes that sum meaningful.

**Keep charge lines in step with the product line.** A buyer who changes the tee's quantity from 144 to 200 in
the cart, or removes it, must not leave run-charge lines behind at 144. `integration-patterns.md`, "Keep
charge lines in step with the product line", has the cart JS that updates or removes the children when the
parent changes and hides the quantity controls on charge lines.

**Drop lines the customer didn't pick.** If a location isn't enabled, drop all its decoration lines. If the
customer wants no PMS match, drop the `pmsMatchVariantId` line. Don't send zero-quantity lines.

---

## 7. Edge cases and rules

- **`priceIncludes == true`**: show the decoration, skip its setup and run lines (extras and PMS may still
  apply).
- **`maxDecoration == 0`**: location exists but decoration isn't allowed for this product; skip it.
- **Absent `*VariantId`**: first fall back to the canonical product's `default_variant_id` (section 4); if
  still absent, the charge truly doesn't apply. Never show a stale price.
- **Both `setupFeeVariantId` and `setupTierVariantId` present**: use the setup tier ladder and ignore the
  flat fee; the ladder is the exact schedule and the flat fee is its base.
- **Personalization decorations**: names ending `- Personalization` collapse to the same canonical method as
  the base ("Heat Transfer - Personalization" becomes "Heat Transfer"). When the supplier publishes distinct
  personalization pricing, the canonical product has a separate variant and enrichment picks it by hash. The
  theme just consumes whatever `*VariantId` it's handed.
- **Whole decoration unenriched** (every `*VariantId` absent on a method the catalog clearly covers): almost
  always the physical product is **missing `psrestful.extra_id`**. Enrichment needs `extra_id` to fetch
  per-product pricing from PSRESTful; without it, it writes nothing rather than guess. Fix: backfill
  `extra_id` in PromoSync (re-import or Update Metafields) and run enrichment again.
- **Unmatched method**: if PromoSync couldn't classify a decoration to a canonical method (rare; usually a
  brand-new method name), every `*VariantId` is absent. Treat as "available but uncharged" or hide. The
  enrichment result email lists the reason; see `notifications.md`.
- **Idempotent re-runs**: enrichment is safe to run repeatedly; identical data is skipped silently. Theme
  code should be resilient to the metafield being byte-identical across renders.

---

## 8. Cross-checking

If a field name or value here disagrees with what you see in a live store, trust the live metafield and check
https://promosync-docs.psrestful.com for the current shape.

See also: `metafields.md` (the canonical-product and enriched-shape catalog entries),
`integration-patterns.md`, "Decoration pricing (enriched `location_decorations`)", for adapt-ready JS, and
`theme-blocks.md` (why the shipped block doesn't price decorations).
