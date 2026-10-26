# Design Coordination: PromoSync + Shopify MCP + Design Tools

This skill is a **PromoSync expert**. It coordinates with two other capabilities:

1. **Shopify MCP / Shopify docs** — for platform-level knowledge (Liquid filters, theme architecture, section schemas, Shopify Function API, cart attributes).
2. **Design tools (nano banana pro, Figma, Midjourney, etc.)** — for visual mockups.

The goal: a user should be able to say *"Design me a beautiful product page for our PromoSync-powered store"* and end up with mockups that already reflect the real data shapes and states.

---

## The three-tool workflow

```
User request
     │
     ▼
┌─────────────────────────┐
│  PromoSync expert       │  ← defines data shape, states, constraints
│  (this skill)           │
└───────────┬─────────────┘
            │ produces design brief
            ▼
┌─────────────────────────┐     ┌──────────────────────────┐
│  Design tool            │────▶│  Shopify MCP             │
│  (nano banana pro)      │     │  (theme conventions,     │
│  produces visual mockup │     │   Liquid, a11y guidance) │
└───────────┬─────────────┘     └──────────────────────────┘
            │
            ▼
┌─────────────────────────┐
│  PromoSync expert       │  ← turn mockup into Liquid + JS
│  + Shopify MCP          │      using real metafield reads
└─────────────────────────┘
```

Your role (as this skill): translate PromoSync's **data reality** into **design constraints** the design tool can work with, then later translate the mockup back into working theme code that reads the actual metafields.

---

## Step 1 — Build the design brief before calling the design tool

Before asking nano banana pro for a mockup, produce a **short, structured brief** that encodes:

- **Component purpose** (e.g., "tier pricing table for product page")
- **Data shape** (from `metafields.md` — paste the relevant JSON example)
- **States to render** (empty, 1 tier, 3 tiers, 5 tiers, matched tier highlighted, variant switching)
- **Copy patterns** (how numbers are formatted, currency, plural/singular)
- **Constraints** (mobile-first, fits alongside native Shopify variant picker, must not duplicate Shopify's native `quantity_price_breaks` UI)
- **Theme context** (reuse the theme's existing design language — `color-{{ color_scheme }}`, `.gradient`, accordion component, price component, product info wrapper spacing)

### Brief template

```
COMPONENT: <name>
PLACEMENT: <page type> — <position relative to other components>

DATA (from PromoSync):
  <metafield path>
  Shape:
    <paste JSON example from metafields.md>
  Notes:
    - <critical rule, e.g. "price in cents">
    - <tier match rule, etc.>

STATES TO DESIGN:
  1. Empty (no data) — <behavior>
  2. Single tier / flat — <behavior>
  3. Multiple tiers / typical — <behavior>
  4. Active/selected state — <behavior>
  5. Variant switch — <transition behavior>

COPY:
  - <exact labels>
  - <pluralization rules>
  - <fallback text>

CONSTRAINTS:
  - Mobile-first (320px → 768px → 1024px)
  - Matches the host theme's design language (see theme.liquid settings)
  - Respects color-scheme CSS variables
  - WCAG AA contrast
  - No conflict with native Shopify <quantity-price-breaks> component

OUT OF SCOPE:
  - Checkout discount UI (applied by Shopify Function, not theme)
  - <anything else>
```

---

## Step 2 — Prompt templates for nano banana pro

Paste one of these after filling in the brief. Keep prompts concrete: specific data values, specific dimensions, specific color tokens.

### Tier pricing table (product page)

```
Design a volume pricing table for a Shopify product page.

DATA (real metafield from PromoSync):
  [
    { "quantityMin": 1,   "price": 2500 },
    { "quantityMin": 10,  "price": 2200 },
    { "quantityMin": 25,  "price": 2000 },
    { "quantityMin": 50,  "price": 1800 },
    { "quantityMin": 100, "price": 1500 }
  ]
  (price is in cents — render as dollars: $25.00, $22.00, $20.00, $18.00, $15.00)

LAYOUTS REQUIRED:
  1. Compact horizontal strip (≤768px width) — 2–3 visible tiers, overflow scrolls
  2. Full table (≥768px) — all tiers, columns: Qty, Price, Save
  3. Collapsible (accordion) variant for use in sidebar

STATES:
  - Default: current tier (based on qty input) highlighted with accent color
  - Hover: row brightens
  - Switching variant: brief fade transition

STYLE:
  - Aesthetic: clean, modern, minimal — matches the host Shopify theme
  - Typography: sans-serif, similar to Inter or system stack
  - Accent color: customer brand blue (placeholder #2563eb)
  - Corners: 8px radius
  - Spacing: generous padding inside cells

OUTPUT: 3 variants showing all three layouts, each in light + dark mode.
```

### Decoration configurator with live pricing (product page)

This is the **v3** decoration UI — it selects locations/methods **and** computes a live price breakdown
(setup + run-tier + extra-color + extra-location + PMS). For the data shape and math see
`decoration-pricing.md`; for the JS see `integration-patterns.md` → "Decoration pricing". Note the shipped
theme block does NOT do pricing — this is a custom-code design.

```
Design a decoration configurator for a promotional product page that shows a live price breakdown.

DATA (real enriched metafield from PromoSync — psrestful.location_decorations):
  [
    { "locationName": "Front Chest", "defaultLocation": true, "decorationsIncluded": 1, "maxDecoration": 1,
      "decorations": [
        { "decorationName": "Screen Print", "maxImprintColors": 6, "default": true, "priceIncludes": false,
          "setupFeeVariantId": "gid://…", "runChargeVariantId": "gid://…",
          "extraColorSetupVariantId": "gid://…", "extraColorRunVariantId": "gid://…",
          "extraLocationSetupVariantId": "gid://…", "extraLocationRunVariantId": "gid://…",
          "pmsMatchVariantId": "gid://…" },
        { "decorationName": "Embroidery", "maxImprintColors": 12, "default": false, "priceIncludes": false,
          "setupFeeVariantId": "gid://…", "runChargeVariantId": "gid://…" }
      ]}
  ]
  Resolved prices (per chosen variant): setup fee $50.00; run charge tier ladder
  [{qty 1: $0.85},{qty 144: $0.70}] (cents in data, render as dollars); extra-color setup $12.00 + run
  $0.10/unit; extra-location setup $68.50 + run $0.05/unit; PMS match $25.00/color.

UX:
  - Location selector (chips or dropdown; default-select defaultLocation).
  - Method selector for that location (default-select the `default` method).
  - Imprint COLOR-COUNT stepper, capped at maxImprintColors. First `includedColors` (usually 1) are free;
    extras add extra-color setup (one-time) + extra-color run (per unit).
  - Optional ADDITIONAL-LOCATION toggle(s); first `decorationsIncluded` are free, extras add
    extra-location setup + run.
  - Optional "Pantone (PMS) match" count — flat per color, per order.
  - Quantity input drives the per-unit terms.
  - LIVE PRICE BREAKDOWN panel itemizing each charge (label, qty basis, line total) and a grand total,
    updating as the customer changes colors / locations / qty.

STATES:
  1. priceIncludes=true method → show method, hide setup/run lines ("Decoration included").
  2. Method with only setup + run (no extras available) → no extra-color/location/PMS rows.
  3. Method with all 7 charges → full breakdown.
  4. Empty / unenriched product (no *VariantId fields) → render selector as info-only, no pricing.
  5. Quantity crosses a run-charge tier boundary (e.g. 143 → 144) → per-unit price drops, breakdown re-totals.

COPY:
  - Prices in cents in data — ALWAYS render as dollars ($0.85, not 85).
  - Distinguish one-time (setup) vs per-unit (run) charges visually.
  - "Extra color" scales with design colors; "PMS match" is the custom-mix cost per Pantone color.

CONSTRAINTS:
  - Sits between variant picker and quantity input (or as an expandable "Customize" panel).
  - ≤600px wide on desktop, full width on mobile.
  - Keyboard-accessible; clear required fields.

OUT OF SCOPE:
  - Tier-price enforcement at checkout (a Shopify Function / draft order does that; the breakdown is display).

OUTPUT: One mobile + one desktop mockup for states 1, 3, and 4.
```

### Minimum quantity notice

```
Design an inline minimum quantity notice for a product page.

DATA: product.metafields.psrestful.minimum_quantity = 25

PLACEMENT: Immediately above the quantity input, below the price.

COPY: "Minimum order: 25 units" — support singular "1 unit" too.

STATES:
  1. Passive notice (min met or not yet interacted)
  2. Error flash (user tried qty below min) — inline error in same spot

STYLE: subtle (gray text + info icon) by default, error state uses warning color.
```

### Volume pricing callout / teaser

```
Design a "Save more in bulk" teaser block for a Shopify product page.

PURPOSE: encourage customers to scroll to the tier pricing table.

DATA INPUT: variant has 3+ tiers in psrestful.part_price_array.
  Compute: "Save up to X% when you buy N or more"
  Example from real data:
    tiers = [1:$25, 10:$22, 25:$20, 50:$18, 100:$15]
    → "Save up to 40% when you buy 100 or more"

STATES:
  1. Has tiers (teaser visible)
  2. No tiers (component hidden)

STYLE: Small horizontal card, accent background, right-pointing CTA.
```

---

## Step 3 — Translating mockups back to Liquid

After the design is approved, come back to this skill + Shopify MCP and:

1. **Identify the data reads** — which metafields does the mockup need? Check `metafields.md`.
2. **Pick the implementation path** — is there an existing PromoSync theme app block (`theme-blocks.md`) that can be configured to match, or do you need custom Liquid?
3. **If custom:** base the implementation on `integration-patterns.md` and existing theme files. Reuse the theme's design tokens (`color-{{ color_scheme }}`, existing component CSS) rather than inventing new ones.
4. **Use Shopify MCP** for platform-specific questions (correct Liquid filter for a use case, section schema for settings, event names for variant change).
5. **Run `shopify theme check`** and test in dev (`shopify theme dev`) before reporting done.

---

## Anti-patterns to avoid

- **Designing before confirming data shape.** Always paste the JSON example from `metafields.md` into the brief. Designers inventing field names leads to mockups that can't be built.
- **Ignoring cents.** Nano banana pro will happily produce `$2500` instead of `$25.00`. The brief must call out "price in cents, render as dollars."
- **Designing the discount.** The theme never applies the discount — the Shopify Function does at checkout. Don't spend design time on discount badges at checkout; focus on the cart preview.
- **Duplicating native Shopify UI.** Shopify has a native `<quantity-price-breaks>` component. Fall back to PromoSync data only when native is empty (see `integration-patterns.md` — "Volume pricing popover").
- **Forgetting the empty state.** Many products won't have tiers/decorations. Every brief must include an empty/absent state.

---

## Quick decision tree

```
User wants a PromoSync-aware UI component.
│
├── Does PromoSync already provide a theme app block for this?
│   └── YES → Recommend configuring the block (theme-blocks.md). Design is for skinning only.
│   └── NO  → Continue ↓
│
├── Is the data already in a PromoSync metafield?
│   └── YES → Reference metafields.md; write a brief with the real JSON shape.
│   └── NO  → Data gap. Surface this to the user — they may need a PromoSync app change, not a theme change.
│
├── Ask design tool for mockups using the brief + prompt template.
│
└── Translate mockup → Liquid using integration-patterns.md + Shopify MCP for platform details.
```
