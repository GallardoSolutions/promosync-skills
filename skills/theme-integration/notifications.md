# PromoSync Notification Emails: What They Mean and Who Fixes It

Read this when a merchant pastes a PromoSync email, when a product or variant that "should be there" is not
in Shopify, or before debugging a storefront display problem. Most "PromoSync data isn't showing" reports
are upstream of the theme: the product was never created, a variant was skipped, or a sync failed and the
merchant got an email about it. Check here first, then go to the Liquid.

PromoSync sends one result email per job to the shop's notification contacts (PromoSync > Settings >
Notification Contacts). Every email ends with support@psrestful.com. Technical "... Errors" emails with
tracebacks go to PSRESTful admins, not merchants, so a merchant will rarely show you one. The messages
below are the merchant-facing wording as of 2026-10-28; the exact sentences matter because they are what
gets pasted.

---

## 1. The emails, by job

| Subject | When it is sent | What to look at |
|---------|-----------------|-----------------|
| **Product Import Results** | After every import run (by product ID, by supplier, or a retry) | Failed products, Skipped products, and "Variants skipped: supplier data issue". A CSV of every row is attached, and PromoSync > Overview > Recent Activity has the same CSV plus a one-click "Retry failed/skipped". |
| **PromoSync Import: Large import started** | A whole-supplier import was queued | Informational. It can take hours; a results email follows. |
| **PromoSync Import: Products Not Found** | Product IDs sent for import do not exist in PSRESTful for that supplier | The IDs are wrong, belong to another supplier, or the supplier has not published them. Nothing to fix in the theme. |
| **PromoSync Import: PSRESTful daily rate limit reached** | The account's daily API quota ran out mid-import | Wait for the next day or upgrade the PSRESTful plan. The run resumes on retry. |
| **Price Sync Results** | After Update Prices (scheduled or manual) | Failed updates, Skipped products, and "Price decreases held" when the shop has Only Increase Prices on. |
| **Inventory Update Results** | After an inventory sync, per location | Failed products, and Skipped meaning there was nothing to update. |
| **Add New Variants Results** | After "Add new variants" for products whose supplier added parts | New variants added, variants skipped because the SKU already exists on the product, and variants skipped because the SKU or part ID already exists elsewhere in the store. |
| **Metafield Update Results** | After Update Metafields | Warnings (updated but not everything applied) and Failed products. |
| **Decoration Products Results** | After "Insert decorations" for a supplier | Decorations skipped (not enough supplier data to publish) and decorations that could not be created. |
| **Location Decorations Enrichment Results** | After the decoration enricher runs | Decorations unmatched, with a reason code. Unmatched means no `*VariantId` was stamped, so the theme cannot price that decoration. See `decoration-pricing.md`. |
| **Add Missing Images Results** | After "Add missing images" | Images added, variants linked, products skipped (no supplier media or nothing missing). |
| **PromoSync: Purchase Order Results** | After orders were turned into purchase orders | Per order: PO created, error, or items skipped. |
| **PO Not Submitted to Supplier** | A PO was created in PSRESTful but could not be forwarded to the supplier | The payload is in the email so it can be sent by hand. Not a theme issue. |
| **PSRESTful API Key Required** | The shop has no PSRESTful private key | Nothing works until the key is pasted into PromoSync > Settings. |

---

## 2. Import errors and skips

These appear under "Failed products" or "Skipped products" in Product Import Results, and the same
translations are reused by Price Sync Results. The first column is the merchant-facing sentence.

### Errors that the merchant or their developer fixes

| Message the merchant sees | Cause | Who fixes it, and where |
|---------------------------|-------|-------------------------|
| "Shopify rejected this product because [a metafield definition] in your store is limited to specific product categories, and this product's category is not one of them. In Shopify admin, go to Settings > Custom data > Products, open it, remove the selected product categories so it applies to all products, save, then retry the import." | Someone restricted a `psrestful.*` metafield definition to certain product categories. Shopify then refuses the whole product. Every import fails the same way until it is undone. | **Developer or merchant, in Shopify admin.** Exactly what the message says. PromoSync writes its metafields on every product it imports, so no definition may be category-restricted. |
| "The supplier rejected your supplier credentials, so this product could not be read. This is not your PSRESTful plan and not a problem with the product. Check the username and password saved for this supplier in PSRESTful ..., test them in Data Explorer, then retry." | The supplier's PromoStandards credentials saved in PSRESTful are wrong or expired. | **Merchant, in PSRESTful** (Configure credentials, then Data Explorer to test). Not Shopify, not the theme. |
| "Too many variants, Shopify limits the app" | The product exceeds Shopify's variant limit for apps. | **Merchant, in PromoSync settings.** Split by color or use a variant grouping strategy; see `metafields.md` "Shop-Level Configuration". |
| "Duplicate product handle, potentially same product imported twice" | A product with the same handle already exists, usually an earlier import or a manual product. | **Merchant or developer, in Shopify admin.** Delete or rename the duplicate, then retry. |
| "Product tags are not valid. Try without importing tags." | A supplier tag breaks Shopify's tag rules. | **Merchant, in PromoSync settings.** Turn off tag import for that supplier and retry. |
| "Product not found in PSRESTful or Supplier" / "Product not found at the supplier. The supplier reports this product ID does not exist or has been discontinued." | Wrong ID, or the supplier discontinued it. | **Merchant.** Verify the ID with the supplier. If it is in the store already, it will stop syncing; consider unpublishing it. |
| "Unauthorized / Forbidden, please check your credentials or upgrade plan, free plan only allows 10 API calls per day" | PSRESTful API key missing or wrong, or the plan does not allow the call. | **Merchant, in PromoSync settings and PSRESTful.** |

### Errors where the right action is "wait and retry"

| Message the merchant sees | Cause | Action |
|---------------------------|-------|--------|
| "Rate limit exceeded, please try again tomorrow or upgrade your PSRESTful plan" | The account's daily PSRESTful quota. | Retry tomorrow, or upgrade. Recent Activity has the retry button. |
| "The supplier is limiting how fast its data can be requested. This is a temporary limit on the supplier's side, not your PSRESTful plan. PromoSync waited and retried ..." | The supplier throttles calls per minute (Showdown Displays, about 40 per minute). | Retry in a few minutes. |
| "Shopify API rate limit reached during import. The system retried automatically but the limit persisted." / "Daily variant creation limit reached." / "This product is currently being modified." | Shopify-side throttling. | Import fewer products at once, or retry later. |
| "This product couldn't be retrieved because one of the supplier's data services is not currently available through PSRESTful." | A PromoStandards service (PPC, Inventory, Media) is not offered or not configured for that supplier. The product itself exists. | Retry shortly; contact PSRESTful support if it persists. Do not tell the merchant to verify the product ID. |
| "A temporary system issue interrupted this import before it finished. No action is needed on your end." | A dropped database connection mid-run. | Retry in a few minutes. |
| "The supplier's system was slow or temporarily unavailable, so this product couldn't be retrieved in time." | Supplier timeout or gateway error. | Retry shortly. |
| "Internal server error, please try again later" | A 500 from PSRESTful or the supplier. | Retry; support if it repeats. |

### Errors that go to PSRESTful support

| Message the merchant sees | Cause |
|---------------------------|-------|
| "Issue with images, contact PSRESTful support" | Shopify rejected an image the supplier served. |
| "Issue with options, contact PSRESTful support" | An option value PromoSync generated does not exist on the product. |
| "Query exceeded Shopify cost limit. This product has too many variants or complex data." | Oversized product. Support may import it in smaller batches. |
| "An unexpected error occurred while importing this product. Our technical team has been notified and will investigate the issue." | Anything not classified above. Admins already received the traceback. |

### Skips (the product was not created, on purpose)

| Message the merchant sees | Meaning | What to tell a developer |
|---------------------------|---------|--------------------------|
| "Product has no active parts or does not meet import criteria" | No sellable parts after the shop's import filters. | Nothing to show; do not look for a theme bug. |
| "The supplier published no price above $0 for this product, so it was not created (it would have been unsellable or listed at $0.00)" | No sellable price in the supplier feed. | Same. If the merchant insists, the supplier must publish a price. |
| "N variants skipped because SUPPLIER is missing the Color attribute on these parts: ... The rest of the product was imported. Please report this data issue to SUPPLIER." | The product imported, but some parts lack a required option value. | The missing variants are a supplier data gap, not a theme or import bug. The merchant reports it to the supplier. |

---

## 3. Price sync

| What the email says | Meaning | Action |
|---------------------|---------|--------|
| "Price decreases held. Only Increase Prices is on: N variants kept their current price because the supplier's price went down. Cost per item was still updated." | The shop chose never to lower prices automatically. | Expected. If the merchant wants the lower price, they turn the setting off or edit the variant. The tier table in `part_price_array` still reflects the held price. |
| Skipped: "No PSRESTful product metafield" / "No extra_id metafield" | The product was not created by PromoSync, or its link metafields were removed. | PromoSync cannot sync it. Re-import it, or leave it as a manual product. A theme reading `psrestful.*` on it gets nothing. |
| Skipped: "Product is discontinued" | The supplier discontinued it. | Prices no longer update. Consider unpublishing. |
| Failed with any message from section 2 | Same translations as import. | Same actions. |

The attached CSV can be uploaded to PromoSync > Targeted Sync > From CSV to re-run exactly the failed and
skipped rows.

---

## 4. Inventory, variants, metafields, images

| Email | What the email says | Meaning |
|-------|---------------------|---------|
| Inventory Update Results | "Skipped (nothing to update)" | Stock already matched. Normal. |
| Inventory Update Results | Failed with a message from section 2 | Usually the supplier's Inventory service was unavailable or the product lost its link metafields. |
| Add New Variants Results | "Skipped Variants (SKU already exists)" | The product already has a variant with that SKU. The merchant reviews whether to rename or re-link the existing variant. Creating it would duplicate a SKU. |
| Add New Variants Results | "Skipped (already in your store)" with "Matched key: sku / part_id / supplier key" | Another product in the store already owns that SKU or part. Nothing is created to avoid a duplicate product. The merchant decides which product should own it. |
| Add New Variants Results | "New Products Created (Missing Colors)" | On Split-by-Color shops, a new color becomes a new product. Expect a new handle in collections. |
| Metafield Update Results | "Warnings (updated, but not everything could be applied)" | The product was updated but one metafield write was refused, most often the category-restricted definition from section 2, or the storefront access flag on a definition. |
| Metafield Update Results | "Skipped (no extra_id)" | Not a PromoSync product. See price sync. |
| Add Missing Images Results | "Skipped (no extra_id, no supplier media, or nothing missing)" | Normal. Only products with supplier media and missing images are touched. |

---

## 5. Decorations

| Email | What the email says | Meaning for decoration pricing |
|-------|---------------------|--------------------------------|
| Decoration Products Results | "Decorations We Skipped: The supplier listed these decorations but didn't give us enough information to publish them." | No canonical decoration product exists for that method. The theme will find no `*VariantId` for it in `location_decorations`, so it cannot be priced. Treat it as "charge does not apply" per `decoration-pricing.md`. |
| Decoration Products Results | "Decorations We Couldn't Create: The supplier returned data we couldn't turn into a Shopify product. Our team has been notified." | PSRESTful support case. |
| Location Decorations Enrichment Results | Unmatched with reason `no_canonical_product` | "Insert decorations" has not been run for that supplier or method. Run it, then re-enrich. |
| Location Decorations Enrichment Results | Unmatched with reason `no_extra_id` | The product is not a PromoSync product, so the enricher cannot look up its pricing. |
| Location Decorations Enrichment Results | Unmatched with reason `no_pricing_hashes` or `no_pricing_for_method` | The supplier publishes no decoration pricing for that method on this product. The theme shows the method without a price. |
| Location Decorations Enrichment Results | Unmatched with reason `hash_mismatch` | The decoration pricing changed after the canonical product was created. Re-run "Insert decorations" to refresh the canonical variants, then re-enrich. |

---

## 6. Purchase orders

| Email | What the email says | Meaning |
|-------|---------------------|---------|
| Purchase Order Results | Skipped item: "No supplier code" / "No ps_product_id" / "No part_id" | The line item is not a PromoSync variant, or lost its metafields (`psrestful.supplier_code`, `product_id`, `part_id`). A theme that lets customers add custom lines must keep those variant metafields intact. |
| Purchase Order Results | Skipped item: "Supplier X not active or not configured" | The merchant has not enabled PO sending for that supplier in PromoSync. |
| Purchase Order Results | Skipped item: "PO Box address incompatible with CARRIER shipping" | The customer entered a PO Box and the shipping method maps to a carrier that cannot deliver there. A checkout validation or a note on the shipping method in the theme avoids this. |
| PO Not Submitted to Supplier | The PO exists in PSRESTful but was not forwarded | Send failure is terminal today; there is no automatic retry. The merchant sends the payload by hand or asks support. |

---

## 7. How the agent should use this

1. When a merchant reports missing storefront data, ask whether they received a PromoSync result email, and
   check Shopify admin for the product and its `psrestful.*` metafields before reading any Liquid.
2. Match the pasted sentence against the tables above and answer with the "who fixes it" column. Do not
   send a merchant to PSRESTful support for a Shopify admin fix, or to Shopify for a credentials fix.
3. For the category-restriction error, walk the developer through Settings > Custom data > Products in the
   Shopify admin; it is the one import failure that is entirely on the Shopify side and blocks everything.
4. Anything not in these tables: point to PromoSync > Overview > Recent Activity for the CSV and retry
   button, then support@psrestful.com with the CSV attached.
