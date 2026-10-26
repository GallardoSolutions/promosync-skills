# PromoSync skills for Claude Code

Skills that teach Claude how to build Shopify themes on top of [PromoSync](https://promosync-docs.psrestful.com), the PSRESTful app that syncs promotional products into Shopify: tier pricing, decorations, minimum quantities, color swatches, blank and sample purchase flows.

## Install

In Claude Code:

```
/plugin marketplace add GallardoSolutions/promosync-skills
/plugin install promosync@psrestful
```

Then open any Shopify theme and ask things like "add volume pricing to the product page" or "price the decorations in the cart". The skill is available as `/promosync:theme-integration`.

## Update

Third-party marketplaces do not auto-update by default. Pull the latest catalog with:

```
/plugin update promosync@psrestful
```

or turn on auto-update for the `psrestful` marketplace in the `/plugin` menu.

## Using it without Claude Code

- **claude.ai**: zip the `skills/theme-integration` folder and upload it as a custom skill.
- **Theme repos**: copy `skills/theme-integration` into your theme at `.claude/skills/theme-integration/` and everyone who clones the theme gets it.

## What is inside

| File | Covers |
|------|--------|
| `SKILL.md` | When to use the skill and the core facts |
| `metafields.md` | Every PromoSync metafield: namespace, key, type, JSON shape |
| `theme-blocks.md` | The theme app blocks and the volume-pricing Shopify Function |
| `integration-patterns.md` | Liquid and JS snippets for product page, cart, and variant switching |
| `decoration-pricing.md` | Pricing decorations from the enriched `location_decorations` metafield |
| `design-coordination.md` | Working with Shopify MCP and design tools on PromoSync UI |

Questions or gaps: https://promosync-docs.psrestful.com or support from PSRESTful.
