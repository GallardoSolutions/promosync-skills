# Theme Workflow for Promotional Products Stores

Read this when the user is starting a store or a theme from scratch, choosing a theme, setting up a local
development loop, or asking how to keep theme code under version control. It is written for distributors and
developers who are new to Shopify theme work. The other files in this skill assume a theme already exists;
this one gets you there safely.

Platform facts here were checked against shopify.dev in October 2026. When a command or menu path does not
match what the user sees, look it up with the Shopify Dev MCP before improvising.

---

## 1. What a promotional products store needs from a theme

PromoSync writes the data. The theme has to show it. A theme is a good fit when it can do all of this:

| Need | Why it matters for promo products | What to check in the theme |
|------|-----------------------------------|----------------------------|
| Online Store 2.0 (JSON templates) | PromoSync ships theme app blocks, which only work in OS 2.0 themes | `templates/product.json` exists, not `product.liquid` |
| App blocks on the product page | Tier pricing, decorations, and the minimum quantity notice are app blocks | The main product section schema lists `{ "type": "@app" }` |
| App blocks in the cart | Volume pricing and decoration charges are shown per line in the cart | The cart section or cart drawer accepts `@app` blocks |
| A variant change event | Tier pricing and the minimum quantity change per variant | Theme fires a DOM event or updates the URL on variant change. Dawn-family themes do; see `integration-patterns.md` |
| A quantity input that is easy to reach | Minimum quantities and price breaks drive the quantity field | The quantity input has a stable selector and is not hidden behind a custom component |
| Room for long option lists | Apparel has many colors and sizes; kits have several part groups | Variant picker handles 50+ values without breaking the layout |
| Fast product pages | Supplier catalogs are large; distributors list thousands of SKUs | Lighthouse score is healthy on a product page with a 20-variant product |

A theme that fails the first two rows is a non-starter. The rest are things to fix with custom Liquid.

### Shopify native B2B is a separate thing

Shopify's own B2B features (company accounts, catalogs, quantity rules, volume pricing) need a Shopify Plus plan.
PromoSync does **not** need them. Its tier pricing comes from `part_price_array` metafields plus the
`quantity-price-breaks` Shopify Function, and its minimum quantity comes from `psrestful.minimum_quantity`.
A distributor on a Basic, Shopify, or Advanced plan gets volume pricing through PromoSync without buying Plus.

This also means a theme's **built-in** "volume pricing" or "quantity rules" UI, such as the one in Trade,
reads Shopify's B2B catalogs, not PromoSync's metafields. Do not expect it to show PromoSync tiers. Use the
PromoSync tier-pricing block or the Liquid in `integration-patterns.md` instead.

---

## 2. Which theme to start from

Shopify's own free themes are the safest base. They are kept current with the platform, their source is
public, and the Shopify Dev MCP knows their structure.

| Theme | When to pick it | Notes for PromoSync |
|-------|-----------------|---------------------|
| **Trade** (free, by Shopify) | A wholesale or B2B style store: quick order lists, bulk buying, a catalog feel | Built for wholesale layouts. Its native volume pricing UI reads Shopify B2B, so swap it for the PromoSync block. Same code family as Dawn, so every Dawn snippet in this skill applies. |
| **Dawn** (free, by Shopify) | A retail-looking store, or when you want the most examples and app compatibility | Shopify's OS 2.0 reference theme. The widest tested path for app blocks. Most PromoSync customers run Dawn or a Dawn fork. |
| **Horizon** family (free, by Shopify, 2025 onward) | A new build where you want Shopify's newest architecture (theme blocks, nestable blocks) | Newer, so fewer PromoSync stores run it. Confirm its product and cart blocks accept `@app` before committing, and expect to adapt the variant-change JS. |
| **Skeleton** (what `shopify theme init` clones) | Only for experienced theme developers building from nothing | Minimal on purpose. Not a good choice for a distributor without a developer. |
| A paid Theme Store theme | When the brand needs a specific look the free themes cannot give | Run the checklist in section 1 on the demo store first. Paid themes vary a lot in app block support and variant-change behavior. Budget time to wire PromoSync into custom product templates. |

Recommendation for most distributors: **Trade** if customers mostly buy in bulk and want a quote-sheet feel,
**Dawn** if the store should look like a consumer brand. Both are free, both are Dawn-family, and both work
with every snippet in this skill.

Do not start from a vintage (pre OS 2.0) theme or a theme that was last updated more than a year ago.

---

## 3. Setting up the store, in order

1. **Create the store** and pick the theme from section 2. Keep the theme unpublished until the catalog is in.
2. **Install PromoSync** and connect the supplier credentials. Import a small set of products first
   (one supplier, a few dozen SKUs). The import writes the metafields the theme will read.
3. **Pick three test products** that between them cover: tier pricing with 4 or more tiers, a decorated
   product with several locations, a product with a minimum quantity above 1, and if the shop sells blanks,
   a product with the blank and sample metafields. Keep their handles in the project README. Every theme
   change is checked against these three.
4. **Turn on the PromoSync embed** (`promo-pricing-engine`) in the theme editor under App embeds. Nothing
   else works without it. See `theme-blocks.md`.
5. **Add the app blocks** to the product template in the theme editor: tier pricing, location decorations,
   minimum quantity notice. Position them near the price and the quantity field.
6. **Only now write custom Liquid**, for the parts the blocks do not cover: cart line pricing, decoration
   pricing, blank and sample modes, color swatches, product specs. Use `integration-patterns.md`.
7. **Publish** once the three test products look right on desktop and mobile.

Skipping to step 6 before step 4 is the most common reason "PromoSync data isn't showing".

---

## 4. Local development with Shopify CLI

Work on a copy, never on the live theme. The CLI gives each developer a private development theme on the
store that only they can see.

### One-time setup

```
npm install -g @shopify/cli@latest
shopify version
```

The user needs the store owner account, or a staff or collaborator account with the Themes permission.

### Starting from a theme already on the store

```
mkdir my-store-theme && cd my-store-theme
shopify theme pull --store my-store.myshopify.com
```

The CLI asks which theme to pull. Pick the unpublished copy, not the live one, so later pushes have a safe
target.

### Starting from a Shopify theme on GitHub

```
shopify theme init my-store-theme --clone-url https://github.com/Shopify/dawn.git
cd my-store-theme
```

Swap the URL for `https://github.com/Shopify/trade.git` or the Horizon repo. Without `--clone-url` the
command clones the minimal Skeleton theme.

### The daily loop

```
shopify theme dev --store my-store.myshopify.com
```

This uploads the folder as a development theme and prints a preview URL plus a theme editor URL. CSS and
section changes hot reload. Open the three test products in the preview and keep them open.

Before every push:

```
shopify theme check
```

Theme Check catches broken Liquid, missing translations, and unknown filters. Fix everything it flags.
When the Shopify Dev MCP is available, also run its validate tool on new Liquid; it catches hallucinated
filters and objects that Theme Check does not know about.

### Pushing

```
shopify theme push --unpublished --theme "Feature: decoration pricing"
```

Push to an unpublished theme, review it in the theme editor, then publish from the admin or with
`shopify theme publish`. Avoid `shopify theme push --live` on a store that is taking orders.

### Several stores or environments

Put them in `shopify.theme.toml` at the theme root so nobody types the wrong store:

```toml
[environments.staging]
store = "my-store-staging.myshopify.com"
theme = "123456789"

[environments.production]
store = "my-store.myshopify.com"
```

Then `shopify theme dev -e staging`. Keep the production environment without a `theme` id so a push to it
always asks.

Files that should never go to Shopify (notes, design exports, scripts) go in `.shopifyignore` at the theme
root. The CLI only works on folders that use the standard theme layout: `assets`, `blocks`, `config`,
`layout`, `locales`, `sections`, `snippets`, `templates`.

---

## 5. Keeping theme code in GitHub

Version control is what lets a distributor undo a bad change, see who changed what, and let more than one
person work on the store. Shopify has a first-party GitHub integration, so no custom deploy pipeline is
needed.

### How the integration behaves

- A **branch is connected to a theme** in the theme library. Pushing commits to the branch updates the theme
  within seconds.
- **Edits made in the Shopify admin are committed back** to the branch by Shopify. This covers the theme
  editor (which writes `config/settings_data.json` and the JSON templates), the code editor, and theme apps.
  This cannot be switched off, so the branch and the theme always match.
- A **disconnected branch cannot be reconnected**. Reconnecting adds a new theme. Decide the branch layout
  once.
- The repo must use the standard theme folder layout at its root. No `src` and `dist` folders on a connected
  branch.

### Recommended layout for a distributor

```
main            connected to the PUBLISHED theme
staging         connected to an unpublished theme used for review
feature/*       short-lived branches, each connected to its own unpublished theme while being reviewed
```

One repo per store. If an agency runs several distributor stores from one codebase, keep one repo per store
anyway and share code by copying, because the settings and template JSON differ per store and Shopify
commits them back.

### Connecting

1. Install the Shopify GitHub app on the GitHub account or organization, granting access only to the theme
   repositories.
2. In the Shopify admin, go to **Online Store > Themes > Add theme > Connect from GitHub**.
3. Pick the repository and the branch. The theme appears in the library with the branch name and last commit
   on its card.
4. Make a small change in the theme editor, save, and check that a commit from the Shopify bot appears on the
   branch. If it does, the loop works in both directions.
5. Publish the theme connected to `main`.

### Day-to-day rules

- **Pull before you work.** Shopify may have committed editor changes since the last pull. Working from a
  stale checkout and pushing creates conflicts in `settings_data.json` that are painful to resolve.
- **Treat `config/settings_data.json` and `templates/*.json` as merchant-owned.** Developers should not
  hand-edit them except to add a section or block. Let the theme editor and Shopify's commits manage them.
- **One feature per branch, one pull request per feature.** Review the diff before merging to `main`, because
  merging publishes.
- **Never commit credentials.** Theme Access passwords, API keys, and `.env` files stay out of the repo. Add
  them to `.gitignore` on day one.
- **Tag releases** (`git tag v2026.10.26`) before large changes such as a theme upgrade, so there is a known
  good point to return to.
- **Theme upgrades** from Shopify (a new Dawn or Trade version) go on a branch, get PromoSync blocks and
  custom Liquid re-applied, and are checked against the three test products before merging.

### When the GitHub integration is not an option

Developers without GitHub access can still use the CLI loop in section 4 and push to unpublished themes.
Keep a plain git repository locally and push to any remote. The difference is that editor changes made in
the admin are not captured automatically, so pull the theme with `shopify theme pull` before each session.

---

## 6. Guardrails for less experienced builders

- **Duplicate the live theme before touching it.** Online Store > Themes > the three-dot menu > Duplicate.
  Work on the copy.
- **Do not copy the PromoSync blocks' Liquid into the theme.** The blocks update with the app. Copied Liquid
  does not. Position and style the blocks; write custom Liquid only for what they do not do.
- **Do not hardcode prices, minimums, or decoration lists** in the theme. Everything comes from metafields,
  so a supplier price change flows through without a theme edit.
- **Check the three test products after every change**, on a phone-sized viewport as well as desktop.
- **Keep custom code in its own files.** New snippets go in `snippets/promosync-*.liquid`, new CSS in
  `assets/promosync.css`. A future theme upgrade is then a matter of carrying a few files across.
- **Write down what was changed and why** in the repo README. The next person, or the next AI session, needs
  it.

---

## 7. How the agent should drive this

When a user is at the start of this journey:

1. Ask for the store URL and which theme they are on, or help them choose with section 2.
2. Confirm PromoSync is installed and the embed is on before debugging any display issue.
3. Set up the CLI loop and the GitHub connection before writing Liquid, unless the user says the store already
   has them.
4. Prefer theme app blocks over custom code. Reach for `integration-patterns.md` only for what the blocks do
   not cover.
5. Run `shopify theme check` and, when available, the Shopify Dev MCP validate tool on every Liquid change.
6. Finish each session with a commit on a feature branch and a short note of what to verify in the theme
   editor.
