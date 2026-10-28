# PromoSync Integration Patterns

Battle-tested Liquid and JS snippets for reading PromoSync metafields in theme code. Use these when the theme-provided app blocks don't fit (custom UI, cart-level logic, JSON endpoints, etc.).

Typical theme files where these patterns land:
- `sections/main-cart-items.liquid` — cart line pricing + volume popover
- `snippets/cart-drawer.liquid` — drawer pricing + volume popover
- `sections/main-cart-footer.liquid` — adjusted cart total
- `sections/main-product.liquid` — product-page volume pricing display

---

## Volume pricing (`psrestful.part_price_array`)

### Match a tier for a given quantity

```liquid
{%- liquid
  assign ps_price_breaks = variant.metafields.psrestful.part_price_array.value
  assign matched_tier_price = nil

  if ps_price_breaks.size > 0
    for tier in ps_price_breaks
      if target_quantity >= tier.quantityMin
        assign matched_tier_price = tier.price
      endif
    endfor
  endif
-%}
```

Rules:
- `price` is in **cents** — render with `{{ matched_tier_price | money_without_currency }}`, never divide by 100.
- Tiers are ascending; the **last** match wins.
- `ps_price_breaks` is a Liquid array of objects (the `.value` filter unwraps the JSON metafield).

### Cart line: show discounted unit price when tier matched

```liquid
{%- liquid
  assign ps_breaks = item.variant.metafields.psrestful.part_price_array.value
  assign matched_tier_price = nil
  if ps_breaks.size > 0
    for tier in ps_breaks
      if item.quantity >= tier.quantityMin
        assign matched_tier_price = tier.price
      endif
    endfor
  endif

  assign has_tier_discount = false
  if matched_tier_price and matched_tier_price < item.original_price
    assign has_tier_discount = true
  endif
-%}

{%- if has_tier_discount -%}
  <div class="cart-item__discounted-prices">
    <s class="cart-item__old-price product-option">{{ item.original_price | money }}</s>
    <strong class="cart-item__final-price product-option">
      ${{ matched_tier_price | money_without_currency }}
    </strong>
  </div>
{%- else -%}
  <span class="price">{{ item.original_price | money }}</span>
{%- endif -%}
```

### Volume pricing popover (falls back to PromoSync when Shopify native is empty)

```liquid
{%- liquid
  assign has_vol_pricing = false
  if item.variant.quantity_price_breaks.size > 0
    assign has_vol_pricing = true
  endif

  assign psrestful_price_breaks = item.variant.metafields.psrestful.part_price_array.value
  if psrestful_price_breaks.size > 0 and has_vol_pricing == false
    assign has_vol_pricing = true
  endif
-%}

{%- if has_vol_pricing -%}
  {%- if item.variant.quantity_price_breaks.size > 0 -%}
    {%- comment -%} Shopify native breaks {%- endcomment -%}
  {%- elsif psrestful_price_breaks.size > 0 -%}
    <volume-pricing class="parent-display">
      <ul class="list-unstyled">
        {%- for tier in psrestful_price_breaks -%}
          <li>
            <span>{{ tier.quantityMin }}<span aria-hidden="true">+</span></span>
            <span>${{ tier.price | money_without_currency }}/ea</span>
          </li>
        {%- endfor -%}
      </ul>
    </volume-pricing>
  {%- endif -%}
{%- endif -%}
```

### Adjusted cart total (visual; real discount applied at checkout)

Use this in `main-cart-footer.liquid`. The Shopify Function applies the actual discount at checkout; this just shows the tier-adjusted total in the cart.

```liquid
{%- liquid
  assign adjusted_total = 0
  for item in cart.items
    assign ps_breaks = item.variant.metafields.psrestful.part_price_array.value
    assign tier_match = nil

    if ps_breaks.size > 0
      for tier in ps_breaks
        if item.quantity >= tier.quantityMin
          assign tier_match = tier.price
        endif
      endfor
    endif

    if tier_match and tier_match < item.original_price
      assign item_total = tier_match | times: item.quantity
    else
      assign item_total = item.final_line_price
    endif

    assign adjusted_total = adjusted_total | plus: item_total
  endfor
-%}

<div class="totals">
  <h2 class="totals__total">{{ 'sections.cart.estimated_total' | t }}</h2>
  {%- if adjusted_total != cart.total_price -%}
    <p class="totals__total-value">${{ adjusted_total | money_without_currency }} {{ cart.currency.iso_code }}</p>
  {%- else -%}
    <p class="totals__total-value">{{ cart.total_price | money_with_currency }}</p>
  {%- endif -%}
</div>
```

### Product page: pass all variants' tiers to JS for dynamic switching

```liquid
{% assign variant_price_breaks = '{' %}
{% for variant in product.variants %}
  {% assign pb = variant.metafields.psrestful.part_price_array.value | json %}
  {% assign variant_price_breaks = variant_price_breaks
      | append: '"' | append: variant.id | append: '":' | append: pb %}
  {% unless forloop.last %}
    {% assign variant_price_breaks = variant_price_breaks | append: ',' %}
  {% endunless %}
{% endfor %}
{% assign variant_price_breaks = variant_price_breaks | append: '}' %}

<div id="variant-data"
     data-variant-id="{{ current_variant.id }}"
     data-variants-price-breaks="{{ variant_price_breaks | escape }}">
</div>
```

```js
function updateVariantPricing(newVariant) {
  const el = document.querySelector("#variant-data");
  const data = JSON.parse(el.getAttribute("data-variants-price-breaks"));
  const tiers = data[newVariant.id];
  if (tiers && tiers.length > 0) renderVolumePricing(tiers);
}

function matchTier(tiers, qty) {
  let best = null;
  for (const t of tiers) if (qty >= t.quantityMin) best = t;
  return best;
}
```

---

## Minimum quantity (`psrestful.minimum_quantity`)

### Read in Liquid

```liquid
{%- assign min_qty = product.metafields.psrestful.minimum_quantity | default: 1 -%}
<input
  type="number"
  name="quantity"
  min="{{ min_qty }}"
  value="{{ min_qty }}"
  step="1">
```

### Show a notice above the quantity input

```liquid
{%- if product.metafields.psrestful.minimum_quantity and product.metafields.psrestful.minimum_quantity > 1 -%}
  <p class="product__min-qty-notice">
    Minimum order: {{ product.metafields.psrestful.minimum_quantity }} units
  </p>
{%- endif -%}
```

### Enforcement

The PromoSync Pricing Engine embed block (`MinQuantityHandler`) enforces the minimum on the input client-side. If you're building a custom quantity component, replicate that logic:

```js
const min = parseInt(inputEl.dataset.min || "1", 10);
inputEl.addEventListener("change", () => {
  if (parseInt(inputEl.value, 10) < min) inputEl.value = min;
});
```

---

## Location decorations — display only (`psrestful.location_decorations`)

> These patterns render decoration **names** and write `properties[Location]` / `properties[Decoration]`.
> They do **not** price the decoration. For setup / run / extra-color / extra-location / PMS pricing and
> multi-line cart-add, use the enriched `*VariantId` fields — see "Decoration pricing (v3 enriched
> `location_decorations`)" below and `decoration-pricing.md`.

### Read in Liquid

```liquid
{%- assign loc_decos = product.metafields.psrestful.location_decorations.value -%}
{%- if loc_decos.size > 0 -%}
  {%- for loc in loc_decos -%}
    <h3>{{ loc.locationName }}</h3>
    <ul>
      {%- for deco in loc.decorations -%}
        <li>
          {{ deco.decorationName }}
          {%- if deco.maxImprintColors %} (up to {{ deco.maxImprintColors }} colors){%- endif -%}
          {%- if deco.priceIncludes %} — included{%- else %} — additional charge{%- endif -%}
        </li>
      {%- endfor -%}
    </ul>
  {%- endfor -%}
{%- endif -%}
```

### Cascading dropdowns → cart line item properties

The recommended pattern is to use the **PromoSync Decorations** theme app block with Display mode = `Dropdown selector`. If you must build it yourself:

```liquid
<select name="properties[Location]" required>
  <option value="">Select location</option>
  {%- for loc in product.metafields.psrestful.location_decorations.value -%}
    <option value="{{ loc.locationName | escape }}">{{ loc.locationName }}</option>
  {%- endfor -%}
</select>

<select name="properties[Decoration]" required disabled>
  <option value="">Select decoration</option>
</select>
```

```js
// Expose the full structure to JS
const locationData = {{ product.metafields.psrestful.location_decorations.value | json }};

locationSelect.addEventListener("change", () => {
  const loc = locationData.find(l => l.locationName === locationSelect.value);
  decorationSelect.innerHTML = '<option value="">Select decoration</option>';
  if (!loc) { decorationSelect.disabled = true; return; }
  for (const d of loc.decorations) {
    const opt = document.createElement("option");
    opt.value = d.decorationName;
    opt.textContent = d.decorationName +
      (d.maxImprintColors ? ` (${d.maxImprintColors} colors)` : "") +
      (d.priceIncludes ? "" : " — additional charge");
    if (d.default) opt.selected = true;
    decorationSelect.appendChild(opt);
  }
  decorationSelect.disabled = false;
});
```

The values end up as `properties[Location]` and `properties[Decoration]` on cart lines, visible in admin and order confirmations. Preserve them through cart templates.

---

## Decoration pricing (v3 enriched `location_decorations`)

When PromoSync's decoration **enricher** has run, each decoration entry carries `*VariantId` GIDs pointing
at canonical decoration-product variants. The theme follows them to compute **setup + run + extra-color +
extra-location + PMS** and adds one cart line per applicable charge. Full model + field reference:
`decoration-pricing.md`. Below is the JS skeleton.

**The shipped `location-decorations` block does not do this** — it's custom code. Run charges use the
variant's `psrestful.part_price_array` (integer cents); the rest use the variant's scalar `price`.

### Kind map + handle suffixes

```js
const KIND_FIELDS = [
  'setupFeeVariantId', 'runChargeVariantId',
  'extraColorSetupVariantId', 'extraColorRunVariantId',
  'extraLocationSetupVariantId', 'extraLocationRunVariantId',
  'pmsMatchVariantId',
];
const KIND_HANDLE_SUFFIX = {
  setupFeeVariantId:           '-setup-fee',
  runChargeVariantId:          '-run-charge',
  extraColorSetupVariantId:    '-extra-color-setup',
  extraColorRunVariantId:      '-extra-color-run',
  extraLocationSetupVariantId: '-extra-location-setup',
  extraLocationRunVariantId:   '-extra-location-run',
  pmsMatchVariantId:           '-pms-match',
};
const slugify = s => s.trim().toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
const numericId = gid => Number(gid.split('/').pop());
```

### Resolve a variant: prefer per-product GID, else canonical `default_variant_id`

```js
// Storefront API client `sf(query, vars)` returns parsed JSON.
const defaultCache = new Map(); // key: handle → resolved variant (or null)

async function resolveVariant(dec, field) {
  if (dec[field]) return { gid: dec[field] };           // per-product pricing — preferred

  const handle = slugify(dec.decorationName) + KIND_HANDLE_SUFFIX[field];
  if (defaultCache.has(handle)) return defaultCache.get(handle);

  const data = await sf(`
    query ($handle: String!) {
      productByHandle(handle: $handle) {
        metafield(namespace: "psrestful", key: "default_variant_id") {
          reference { ... on ProductVariant {
            id
            price { amount }
            metafield(namespace: "psrestful", key: "part_price_array") { value }
          } }
        }
      }
    }`, { handle });

  const ref = data?.productByHandle?.metafield?.reference || null;
  const resolved = ref
    ? { gid: ref.id, priceCents: Math.round(parseFloat(ref.price.amount) * 100),
        ladder: ref.metafield ? JSON.parse(ref.metafield.value) : null }
    : null;                                              // no default → charge doesn't apply
  defaultCache.set(handle, resolved);
  return resolved;
}
```

### Tier match + per-kind math

```js
function tierPriceCents(ladder, qty) {
  let chosen = ladder[0];
  for (const t of ladder) { if (t.quantityMin <= qty) chosen = t; else break; }
  return chosen.price;                                   // integer cents
}

// `prices` = { setupFeeVariantId: {priceCents}, runChargeVariantId: {priceCents, ladder}, ... }
// resolved earlier via resolveVariant() (skip any kind that resolved to null).
function decorationTotalCents(prices, { qty, chosenColors, includedColors, chosenLocations, decorationsIncluded, pmsColors }) {
  const extrasColors = Math.max(0, chosenColors - includedColors);
  const extrasLocs   = Math.max(0, chosenLocations - decorationsIncluded);
  const at = f => prices[f] ? prices[f].priceCents : 0;
  let total = 0;
  total += at('setupFeeVariantId');                                          // ×1
  if (prices.runChargeVariantId)
    total += tierPriceCents(prices.runChargeVariantId.ladder, qty) * qty;    // per unit
  total += at('extraColorSetupVariantId')    * extrasColors;                 // one-time
  total += at('extraColorRunVariantId')      * extrasColors * qty;           // per unit
  total += at('extraLocationSetupVariantId') * extrasLocs;                   // one-time
  total += at('extraLocationRunVariantId')   * extrasLocs * qty;             // per unit
  total += at('pmsMatchVariantId')           * pmsColors;                    // flat, per order
  return total;                                                              // cents
}
```

`includedColors` comes from supplier config (default `1`); `decorationsIncluded` is on the location entry.
If `dec.priceIncludes === true`, skip the setup/run terms (extras/PMS may still apply).

### Multi-line cart-add

```js
async function addDecoratedToCart({ physicalVariantId, dec, prices, qty, extrasColors, extrasLocs, pmsColors }) {
  const items = [{ id: physicalVariantId, quantity: qty }];
  const push = (field, quantity, props) => {
    if (prices[field] && quantity > 0) items.push({ id: numericId(prices[field].gid), quantity, ...(props ? { properties: props } : {}) });
  };
  if (!dec.priceIncludes) {
    push('setupFeeVariantId', 1);
    push('runChargeVariantId', qty); // tier price enforced at checkout via cart-transform Function/draft order
  }
  push('extraColorSetupVariantId', extrasColors);
  push('extraColorRunVariantId', extrasColors * qty);
  push('extraLocationSetupVariantId', extrasLocs);
  push('extraLocationRunVariantId', extrasLocs * qty);
  push('pmsMatchVariantId', pmsColors);

  await fetch('/cart/add.js', {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ items }),
  });
}
```

**Caveat:** `/cart/add.js` charges the run-charge variant's headline (qty-1) price. Enforce the real tier
with a cart-transform Shopify Function (read `part_price_array`, override line price) or a draft order. A
pure-Liquid theme shows the correct tier price but charges the headline — reconcile at order time.

---

## Variant gallery (`psrestful.gallery`) — gated feature

**Important:** This metafield is **only present on paid PromoSync plans** (Trial, Business, Enterprise) for shops **not** using the `by_color` variant grouping strategy. On all other shops the metafield is empty/nil and the theme must fall back to native Shopify media.

The design principle:

> Treat `psrestful.gallery` as a progressive enhancement. If it's there, you can give shoppers a true per-variant gallery (e.g., 6 images of the Red variant from every angle, swapping to 6 images of the Blue variant on selection). If it's not, fall back to `product.media` + `variant.featured_media` and the page still looks fine.

Each entry in the list is a Shopify File object — typically a `MediaImage` (use `.image`). The first entry is the variant's featured image (same as `variant.featured_media`); subsequent entries are additional color-matched media, deduped.

### Detect availability + render

```liquid
{%- liquid
  assign vg = variant.metafields.psrestful.gallery.value
  assign has_variant_gallery = false
  if vg.size > 0
    assign has_variant_gallery = true
  endif
-%}

{%- if has_variant_gallery -%}
  {%- comment %} Gated feature available — render per-variant gallery {%- endcomment -%}
  <ul class="product__gallery product__gallery--variant" data-source="psrestful">
    {%- for media in vg -%}
      {%- if media.image -%}
        <li class="product__gallery-item">
          <img
            src="{{ media.image | image_url: width: 1200 }}"
            srcset="{{ media.image | image_url: width: 600 }} 600w,
                    {{ media.image | image_url: width: 1200 }} 1200w,
                    {{ media.image | image_url: width: 2000 }} 2000w"
            sizes="(min-width: 750px) 50vw, 100vw"
            alt="{{ media.image.alt | escape }}"
            loading="lazy"
            width="{{ media.image.width }}"
            height="{{ media.image.height }}">
        </li>
      {%- endif -%}
    {%- endfor -%}
  </ul>
{%- else -%}
  {%- comment %} Fallback: native Shopify media (lead = variant's featured if present) {%- endcomment -%}
  <ul class="product__gallery product__gallery--native" data-source="shopify">
    {%- if variant.featured_media -%}
      <li class="product__gallery-item">
        <img src="{{ variant.featured_media | image_url: width: 1200 }}"
             alt="{{ variant.featured_media.alt | escape }}"
             loading="eager" fetchpriority="high">
      </li>
    {%- endif -%}
    {%- for media in product.media -%}
      {%- if variant.featured_media and media.id == variant.featured_media.id -%}
        {%- continue -%}
      {%- endif -%}
      {%- if media.media_type == 'image' -%}
        <li class="product__gallery-item">
          <img src="{{ media | image_url: width: 1200 }}" alt="{{ media.alt | escape }}" loading="lazy">
        </li>
      {%- endif -%}
    {%- endfor -%}
  </ul>
{%- endif -%}
```

### Variant switching: dump all variants' galleries to JSON

Lets JS replace the gallery instantly on variant change, without an HTTP round-trip. **Skip the JSON dump entirely when no variant has a gallery** — keeps the payload small for free-tier shops.

```liquid
{%- liquid
  assign any_variant_has_gallery = false
  for v in product.variants
    if v.metafields.psrestful.gallery.value.size > 0
      assign any_variant_has_gallery = true
      break
    endif
  endfor
-%}

{%- if any_variant_has_gallery -%}
  {%- capture variant_galleries -%}
    {
      {%- for v in product.variants -%}
        "{{ v.id }}": [
          {%- assign vg = v.metafields.psrestful.gallery.value -%}
          {%- for media in vg -%}
            {%- if media.image -%}
              {
                "url": {{ media.image | image_url: width: 1600 | json }},
                "alt": {{ media.image.alt | default: '' | json }},
                "w": {{ media.image.width }},
                "h": {{ media.image.height }}
              }{%- unless forloop.last %},{% endunless -%}
            {%- endif -%}
          {%- endfor -%}
        ]{%- unless forloop.last %},{% endunless -%}
      {%- endfor -%}
    }
  {%- endcapture -%}
  <script type="application/json" id="ps-variant-galleries">{{ variant_galleries }}</script>
{%- endif -%}
```

```js
// Listen for variant change (Dawn/Trade emit a custom event; adapt as needed).
const galleriesEl = document.getElementById('ps-variant-galleries');
const galleries = galleriesEl ? JSON.parse(galleriesEl.textContent) : null;

function renderVariantGallery(variantId) {
  if (!galleries) return false; // feature not present — let native gallery handle it
  const items = galleries[variantId];
  if (!items || items.length === 0) return false;

  const ul = document.querySelector('.product__gallery');
  if (!ul) return false;

  ul.innerHTML = items.map(it => `
    <li class="product__gallery-item">
      <img src="${it.url}" alt="${it.alt}" width="${it.w}" height="${it.h}" loading="lazy">
    </li>
  `).join('');
  return true;
}

document.addEventListener('variant:change', (e) => {
  const v = e.detail && e.detail.variant;
  if (v) renderVariantGallery(v.id);
});
```

### Common gotchas

1. **Always check `.size > 0` before iterating.** When the gate is closed, `.value` is nil and `.size` is 0; iterating is safe but rendering empty `<ul>` is ugly.
2. **Don't compare against `product.media` count to "detect" the feature.** Use the metafield itself. A free-tier shop with 10 native media files is not the same as a paid-tier shop with 10 variant-gallery entries.
3. **Index 0 of the gallery is already the variant's featured image** — don't render `variant.featured_media` *and* the full gallery, you'll double up. If you want a hero+thumbnails layout, treat `gallery[0]` as the hero and `gallery[1..]` as thumbnails.
4. **`list.file_reference` can technically hold non-image files** (PromoSync currently only writes images, but Liquid doesn't enforce it). Use `if media.image` as a type guard before treating it as an image.
5. **Storefront access must be enabled** on the metafield definition for the theme to read it. If you see no data on a known-paid-tier shop, that's the first thing to check.
6. **`by_color` shops never get this metafield, even on paid tiers** — because each color is already a separate product. Don't surprise the merchant by promising a per-variant gallery; if their grouping strategy is `by_color`, the fallback path is what they'll always see.
7. **Gallery contents are supplier-specific** — the list is built by a `MediaAssociationStrategy` chosen by `supplier_code`. SanMar / S&S cascade through partId → color → fallback color (typically rich); HIT uses partId only (typically sparser); unmapped suppliers run the default cascade. So an "empty gallery on a paid-tier shop" can be legitimate (supplier didn't tag the media that way) rather than a bug. See `metafields.md` → "Media association strategies" for the full table.

---

## Blank / Decorated / Sample purchase flows (gated trio)

**Important:** The blank/sample metafields (`blank_minimum_quantity`, `blank_part_price_array`, `sample_price`) are **only present on shops where `Shop.emit_blank_sample_metafields == true`** — that is, `enable_blank_sample_sales=true` AND `default_configuration_type='Decorated'`. On Blank-default shops the regular `minimum_quantity`/`part_price_array` already *are* the blank values, so the duplicate `blank_*` namespace isn't emitted.

The pattern below lets a theme offer three purchase modes side-by-side on a single product page:

| Mode | Min qty source | Tier pricing source | Single-unit shortcut |
|---|---|---|---|
| **Decorated** (always available) | `product.metafields.psrestful.minimum_quantity` | `variant.metafields.psrestful.part_price_array` | n/a — decoration mins are typically ≥ 6 |
| **Blank** (gated) | `product.metafields.psrestful.blank_minimum_quantity` | `variant.metafields.psrestful.blank_part_price_array` | n/a |
| **Sample** (gated, conditional) | always 1 | n/a — single unit | `variant.metafields.psrestful.sample_price` (cents) |

The Sample mode is only valid when `sample_price` is present — that flag means the supplier offers a qty-1 blank tier. If `sample_price` is nil, do not render the Sample option.

### Feature-detect the gate

```liquid
{%- liquid
  assign dec_min = product.metafields.psrestful.minimum_quantity | default: 1
  assign dec_tiers = variant.metafields.psrestful.part_price_array.value
  assign blank_min = product.metafields.psrestful.blank_minimum_quantity
  assign blank_tiers = variant.metafields.psrestful.blank_part_price_array.value
  assign sample_cents = variant.metafields.psrestful.sample_price

  assign supports_blank = false
  if blank_min or blank_tiers.size > 0
    assign supports_blank = true
  endif

  assign supports_sample = false
  if sample_cents
    assign supports_sample = true
  endif
-%}
```

Use `supports_blank` / `supports_sample` to conditionally render those tabs/buttons. **Never assume the gate is open** — always check.

### Three-mode selector (cards/radios)

```liquid
<fieldset class="config-type-selector" data-config-selector>
  <legend>Choose configuration</legend>

  <label class="config-type-option config-type-option--decorated">
    <input type="radio" name="properties[Configuration]" value="Decorated"
           data-min-qty="{{ dec_min }}" data-mode="decorated" checked>
    <span class="config-type-option__title">Decorated</span>
    <span class="config-type-option__hint">Custom imprint · min {{ dec_min }} units</span>
  </label>

  {%- if supports_blank -%}
    <label class="config-type-option config-type-option--blank">
      <input type="radio" name="properties[Configuration]" value="Blank"
             data-min-qty="{{ blank_min | default: 1 }}" data-mode="blank">
      <span class="config-type-option__title">Blank</span>
      <span class="config-type-option__hint">No imprint · min {{ blank_min | default: 1 }} units</span>
    </label>
  {%- endif -%}

  {%- if supports_sample -%}
    <label class="config-type-option config-type-option--sample">
      <input type="radio" name="properties[Configuration]" value="Sample"
             data-min-qty="1" data-mode="sample" data-sample-price="{{ sample_cents }}">
      <span class="config-type-option__title">Sample</span>
      <span class="config-type-option__hint">
        Single unit, blank · ${{ sample_cents | money_without_currency }}
      </span>
    </label>
  {%- endif -%}
</fieldset>
```

The `properties[Configuration]` line item property is what flows into cart, checkout, order admin, and order emails. Pick whatever convention the merchant prefers (`Mode`, `Type`, etc.); keep it consistent across the theme. Hide the location-decorations block when mode is `Blank` or `Sample` — there's nothing to decorate.

### Per-mode price + min-qty update on selection

```js
const selector = document.querySelector('[data-config-selector]');
const qtyInput = document.querySelector('input[name="quantity"]');
const priceEl  = document.querySelector('[data-product-price]');

// Cached tier arrays for the active variant; refresh on variant change.
let tiers = {
  decorated: {{ dec_tiers | json }},
  blank: {{ blank_tiers | default: '[]' | json }},
  samplePrice: {{ sample_cents | default: 'null' }},
};

function matchTier(arr, qty) {
  let best = null;
  for (const t of (arr || [])) if (qty >= t.quantityMin) best = t;
  return best;
}

selector.addEventListener('change', (e) => {
  const input = e.target;
  if (input.name !== 'properties[Configuration]') return;

  const mode = input.dataset.mode;
  const min  = parseInt(input.dataset.minQty || '1', 10);

  qtyInput.min = min;
  if (parseInt(qtyInput.value, 10) < min) qtyInput.value = min;

  // Recompute and display unit price for the chosen mode.
  let unitCents = null;
  if (mode === 'sample') {
    unitCents = tiers.samplePrice;
  } else {
    const arr = mode === 'blank' ? tiers.blank : tiers.decorated;
    const t = matchTier(arr, parseInt(qtyInput.value, 10));
    if (t) unitCents = t.price;
  }
  if (unitCents != null) {
    priceEl.textContent = '$' + (unitCents / 100).toFixed(2);
  }
});
```

(Liquid → JS price math is shown here for clarity. If you want server-side accuracy, render per-mode prices in Liquid and just toggle visibility on selection.)

### Cart line: render the right price per configuration

The Shopify Function (`quantity-price-breaks`) only knows about `psrestful.part_price_array`. **It does not apply the blank or sample price at checkout.** For Blank/Sample line items, the merchant must use a different mechanism (manual price override at order processing, or a separate Shopify Function variant — outside scope of this skill).

For the theme's *visual* cart display, branch on the line item's `Configuration` property:

```liquid
{%- liquid
  assign cfg = item.properties['Configuration']
  assign unit_cents = nil

  if cfg == 'Sample'
    assign unit_cents = item.variant.metafields.psrestful.sample_price
  elsif cfg == 'Blank'
    assign blank_tiers = item.variant.metafields.psrestful.blank_part_price_array.value
    for tier in blank_tiers
      if item.quantity >= tier.quantityMin
        assign unit_cents = tier.price
      endif
    endfor
  else
    {%- comment %} Decorated (default) — use the standard tier match {%- endcomment -%}
    assign dec_tiers = item.variant.metafields.psrestful.part_price_array.value
    for tier in dec_tiers
      if item.quantity >= tier.quantityMin
        assign unit_cents = tier.price
      endif
    endfor
  endif
-%}

{%- if unit_cents -%}
  <strong class="cart-item__unit">${{ unit_cents | money_without_currency }}/ea</strong>
  {%- if cfg and cfg != 'Decorated' -%}
    <span class="cart-item__config-badge cart-item__config-badge--{{ cfg | downcase }}">{{ cfg }}</span>
  {%- endif -%}
{%- else -%}
  <span class="price">{{ item.original_price | money }}</span>
{%- endif -%}
```

### Sample-mode-specific UX

- **Force quantity to 1 and disable the qty input** when the Sample radio is selected — anything > 1 isn't Sample, it's Blank-tier-1.
- **Render `sample_price` directly on the selector card** (as in the radio markup above) so the customer sees the exact cost before clicking.
- **Hide volume pricing UI** (tier table, "save more in bulk" callouts) when Sample is selected — irrelevant for a single-unit order.
- **Add a short helper string** like "Single-unit blank for evaluation — no minimum, no decoration" so customers understand what Sample means in this context.

### Gotchas

1. **Always feature-detect both gates separately.** `supports_blank` does not imply `supports_sample` — a product can have a blank tier array starting at qty 6, in which case blank is offered but sample is not. The codebase enforces this: `sample_price` is only written when `blank_part_price_array`'s lowest `quantityMin == 1`.
2. **Don't fabricate a sample price** from the cheapest blank tier when `sample_price` is nil. If the supplier didn't offer qty-1, the merchant can't fulfill it.
3. **`part_price_array` semantics differ across shops.** On a Decorated-default shop with the gate open, `part_price_array` = decorated tiers and `blank_part_price_array` = blank tiers. On any other shop, `part_price_array` already *is* the blank tiers and `blank_part_price_array` is absent. Theme code that simply reads `part_price_array` works correctly in both cases for the default mode — but blank/sample tabs must check for the metafield's existence before rendering.
4. **The Shopify Function (`quantity-price-breaks`) does not apply blank or sample prices at checkout.** It only reads `psrestful.part_price_array`. If the merchant needs Blank/Sample lines to actually charge at those prices, that's an order-processing concern outside the theme. The theme's role is *display* + *line item property tagging*.
5. **Preserve `properties[Configuration]`** through cart, checkout, and order. Order admin and order confirmation emails should show which configuration was purchased.
6. **Decorations block must be hidden in Blank/Sample mode.** Selecting a blank with `properties[Location]=Front Chest` is a contradiction — guard against it in JS by listening for the configuration change and clearing/disabling those selects.

---

## Supplier / brand / classification metafields

```liquid
{%- assign supplier = product.metafields.psrestful.supplier_code -%}
{%- assign brand = product.metafields.psrestful.brand -%}
{%- assign material = product.metafields.psrestful.primary_material -%}
{%- assign lead_time = product.metafields.psrestful.lead_time -%}

{%- if lead_time -%}
  <p class="product__lead-time">Ships in ~{{ lead_time }} business days</p>
{%- endif -%}
```

---

## Cautions and compliance flags

```liquid
{%- if product.metafields.psrestful.is_caution -%}
  <div class="product__caution" role="alert">
    {{ product.metafields.psrestful.caution_comment | newline_to_br }}
  </div>
{%- endif -%}

{%- if product.metafields.psrestful.is_hazmat -%}
  <span class="badge badge--hazmat">Hazardous material</span>
{%- endif -%}
```

---

## Color swatches (option-linked metaobjects)

PromoSync links the **Color option** to per-color `custom-color-swatch` metaobjects
(`label` + `hex_color`) and exposes them on the product as
`psrestful.color-swatches` (`list.metaobject_reference`). See `metafields.md` →
"Color swatches" for the full write pipeline. The theme side has two routes:

**Route A — native picker.** If your theme already renders Shopify's linked-option
swatches, nothing to do: the option is linked, swatches appear. But because the
metaobject is a **custom** type (not Shopify's standard color/pattern), the native
`option_value.swatch` object is **not** auto-populated — so Dawn-style
`{{ value.swatch.color }}` will be empty. Use Route B for a custom theme.

**Route B — read the metafield directly.** Build a name→hex map from the metaobject
list and match it to each Color option value by `label`.

> ⚠️ **Hyphenated key:** `psrestful.color-swatches` must use bracket access —
> `product.metafields.psrestful["color-swatches"]` — or Liquid reads the hyphen as
> minus (`color - swatches`).

```liquid
{%- liquid
  assign swatches = product.metafields.psrestful["color-swatches"].value
  assign swatch_labels = '' | split: ''
  assign swatch_hexes  = '' | split: ''
  for swatch in swatches
    assign swatch_labels = swatch_labels | push: swatch.label
    assign swatch_hexes  = swatch_hexes  | push: swatch.hex_color
  endfor
-%}

{%- for option in product.options_with_values -%}
  {%- assign oname = option.name | downcase -%}
  {%- if oname == 'color' or oname == 'colour' -%}
    <div class="color-swatches" role="radiogroup" aria-label="{{ option.name }}">
      {%- for value in option.values -%}
        {%- comment -%} Find this value's hex by matching label {%- endcomment -%}
        {%- assign hex = '' -%}
        {%- for label in swatch_labels -%}
          {%- if label == value.name -%}
            {%- assign hex = swatch_hexes[forloop.index0] -%}
            {%- break -%}
          {%- endif -%}
        {%- endfor -%}
        <label class="color-swatch" title="{{ value.name | escape }}">
          <input type="radio" name="Color" value="{{ value.name | escape }}"{% if forloop.first %} checked{% endif %}>
          {%- if hex != blank -%}
            <span class="color-swatch__dot" style="background:{{ hex }};"></span>
          {%- else -%}
            {%- comment -%} No swatch metaobject for this color — fall back to a label {%- endcomment -%}
            <span class="color-swatch__name">{{ value.name }}</span>
          {%- endif -%}
        </label>
      {%- endfor -%}
    </div>
  {%- endif -%}
{%- endfor -%}
```

**Why match by `label`, not index:** option values and the metaobject list are not
guaranteed to share order, and a color may legitimately have **no** metaobject (no
hex from the supplier → skipped server-side). Always have the no-hex fallback —
don't assume every color value has a swatch.

> For **this theme (kantvas)** specifically: the PDP currently fakes swatches with
> `style="background: {{ value | downcase | replace: ' ', '' }}"` (a raw CSS-color-name
> guess that fails for names like "Monterey Sage"). Replacing that with the snippet
> above — gated on `product.metafields.psrestful["color-swatches"].value.count > 0`
> with the CSS-name guess as the fallback — is the correct upgrade. Confirm the
> store actually writes `color-swatches` first (it uses a bespoke `psrestful` schema).

---

## Marketing points (product specs) — `psrestful.marketing_points`

Raw supplier `ProductMarketingPointArray` (`[{pointType, pointCopy}]`). It is **not
normalized**, so read `metafields.md` → "`marketing_points` (product)" for the per-supplier
survey first. Always filter with an **allow-list** of `pointType`s. Never loop and print everything:
PCNA alone would dump ~98 internal attributes per product.

```liquid
{%- assign points = product.metafields.psrestful.marketing_points.value -%}
{%- if points and points.size > 0 -%}
  {%- comment -%}
    Customer-facing bullet types differ per supplier:
    Bullet Point (Peerless, Stromberg, Diamondback, WOWLine…), Feature (Primeline, Ariel,
    Gemline, Logomark), Highlight (HIT), BTH (Koozie Group). Extend after sampling a supplier's data.
    Untyped points (pointType null) are Showdown's one-line tagline, rendered as an intro.
  {%- endcomment -%}
  {%- assign bullet_types = 'Bullet Point,Feature,Highlight,BTH' | split: ',' -%}
  {%- assign spec_types = 'Material,Product Width (IN),Product Length (IN),Product Height (IN),Product Weight (LB)' | split: ',' -%}

  {%- capture tagline -%}
    {%- for point in points -%}
      {%- if point.pointType == blank -%}{{ point.pointCopy | escape }}{%- break -%}{%- endif -%}
    {%- endfor -%}
  {%- endcapture -%}

  {%- capture bullets -%}
    {%- for point in points -%}
      {%- if bullet_types contains point.pointType -%}
        {%- comment -%} Koozie's "Top markets: …" lines are distributor targeting {%- endcomment -%}
        {%- assign copy_start = point.pointCopy | slice: 0, 12 | downcase -%}
        {%- unless copy_start == 'top markets:' -%}
          <li>{{ point.pointCopy | escape }}</li>
        {%- endunless -%}
      {%- endif -%}
    {%- endfor -%}
  {%- endcapture -%}

  {%- capture specs -%}
    {%- for point in points -%}
      {%- if spec_types contains point.pointType -%}
        <tr><th scope="row">{{ point.pointType | escape }}</th><td>{{ point.pointCopy | escape }}</td></tr>
      {%- endif -%}
    {%- endfor -%}
  {%- endcapture -%}

  {%- if tagline != blank -%}
    <p class="product-tagline">{{ tagline }}</p>
  {%- endif -%}
  {%- if bullets != blank -%}
    <ul class="product-points">{{ bullets }}</ul>
  {%- endif -%}
  {%- if specs != blank -%}
    <table class="product-specs">{{ specs }}</table>
  {%- endif -%}
{%- endif -%}
```

### Gotchas
- **Absent is normal.** Many suppliers send nothing (BEL Promo, Magnet Group, Gold Bond,
  Evans, Tekweld, …), and HIT/Logomark/Showdown fill it only for some products. Render nothing,
  or fall back to `product.description`. Don't show an empty "Specifications" tab.
- **Use `capture` + `!= blank`** as above, so a product whose points are all filtered out
  doesn't render an empty `<ul>`/tab heading.
- **Never print 0/1 flag types** (`Eco-Friendly`, `Woman Owned`, `PromoCares Collection`, …).
  `0` means "no".
- **Hide `Additional Info`, `Industry`, `Price`, `NOTE`, `Compliance`, `HTS *`** by default.
  They're distributor notes, target markets, compliance and customs data.
- **`pointType` can be `null`** (Showdown Displays). Verified 2026-10-02: Showdown always sends
  exactly one untyped point, a ~100-char marketing tagline. That's why the snippet renders it as
  an intro `<p>`, not a bullet. `contains` never matches `null`, so it can't leak into the list.
- **Koozie Group's `BTH` = real bullets** (verified 2026-10-02), but ~3% start with
  `Top markets:` (target markets for distributors), and the snippet skips those. A few mention
  option charges ("Optional magnet available for an additional charge"). That's acceptable copy,
  but it can conflict with how the store prices options.
- **Liquid only.** The definition isn't storefront-exposed, so client-side Storefront API
  fetches return null. Render server-side.
- **Imported before 2026-09-30?** The product won't have it until the merchant runs
  **Update Metafields** on it.
- Unit labels live inside `pointType` (`Product Width (IN)`). Strip the suffix with
  `| remove: ' (IN)'` if you want a cleaner label.

---

## Shipping weight and carton info

PromoSync writes a per-unit shipping weight onto every variant's inventory item (see `metafields.md` →
"Shipping weight"), so Shopify's weight-based and carrier-calculated rates already work. These snippets only
*display* that data. Never recompute shipping weight in the theme.

### Estimated shipping weight per cart line

`item.variant.weight` is always in grams; `weight_with_unit` formats it in the variant's unit.

```liquid
{%- comment -%} Inside the cart line-item loop. Hidden when PromoSync could not determine a weight. {%- endcomment -%}
{%- if item.variant.weight > 0 -%}
  {%- assign line_weight = item.variant.weight | times: item.quantity -%}
  <span class="cart-item__weight">
    Est. shipping weight: {{ line_weight | weight_with_unit: item.variant.weight_unit }}
  </span>
{%- endif -%}
```

### Total estimated weight in the cart footer

```liquid
{%- liquid
  assign cart_weight = 0
  for item in cart.items
    assign line_weight = item.variant.weight | times: item.quantity
    assign cart_weight = cart_weight | plus: line_weight
  endfor
-%}
{%- if cart_weight > 0 -%}
  <p class="cart__weight">Estimated shipping weight: {{ cart_weight | weight_with_unit }}</p>
{%- endif -%}
```

`cart.total_weight` gives the same number in grams without the loop; the loop is shown so it can skip lines
you do not want counted (for example decoration charge lines, see `decoration-pricing.md`).

### "Ships N per carton" on the product page

Reads the master carton from `shipping_package_array`. Values are strings, so coerce with `plus: 0`.

```liquid
{%- liquid
  assign cartons = variant.metafields.psrestful.shipping_package_array.value
  assign carton_qty = 0
  assign carton_weight = 0
  assign carton_uom = ''
  for pack in cartons
    assign qty = pack.quantity | plus: 0
    if qty > carton_qty
      assign carton_qty = qty
      assign carton_weight = pack.weight | plus: 0
      assign carton_uom = pack.weightUom | downcase
    endif
  endfor
-%}
{%- if carton_qty > 0 -%}
  <p class="product__carton">
    Ships {{ carton_qty }} per carton{% if carton_weight > 0 %}, about {{ carton_weight }} {{ carton_uom }} per carton{% endif %}
  </p>
{%- endif -%}
```

Pair this with the minimum quantity notice: distributors often want to nudge buyers toward full-carton
quantities, and `carton_qty` is the number to suggest.

### Product dimensions row

```liquid
{%- assign dim = variant.metafields.psrestful.dimension.value -%}
{%- if dim and dim.width and dim.height -%}
  <p class="product__dimensions">
    {{ dim.width }} x {{ dim.height }}{% if dim.depth %} x {{ dim.depth }}{% endif %} {{ dim.dimensionUom | downcase }}
    {%- if dim.weight and dim.weight != "0" %}, {{ dim.weight }} {{ dim.weightUom | downcase }}{% endif %}
  </p>
{%- endif -%}
```

## Debugging tips

1. **Metafield returns nothing?** Check that storefront access is enabled on the metafield definition in Shopify admin. PromoSync may write the value, but `product.metafields.*` won't expose it to the storefront unless the definition allows it.
2. **Use `| json` in Liquid to dump a value to HTML for inspection:** `<script>window.__ps_debug = {{ product.metafields.psrestful.location_decorations.value | json }};</script>`
3. **Check write status via admin:** `Products → [product] → Metafields` — should show `psrestful.*` entries.
4. **`.value` matters for JSON metafields:** `product.metafields.psrestful.location_decorations` (raw metafield object) vs `.value` (parsed JSON). Always use `.value` to iterate.
5. **Run `shopify theme check` after every Liquid change.**
