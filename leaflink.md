# Distru onboarding playbook: LeafLink

Part of the Distru onboarding guide for AI agents: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md. Verified end to end on a 35-product edibles menu with 43 photos. The mapping was also dry-run on a flower, pre-roll, and vape menu.

## 1. Find the menu

- **The user gave a URL** such as `https://app.leaflink.com/menu/acme/`. The slug is `acme`.
- **The user gave only a brand name.** Web search "<brand> site:app.leaflink.com/menu", or try the brand name lowercased with dashes as the slug.
- **Sister brands** are listed under `company.additional_brands` (e.g. Bluebird `bluebird-7`, Fernwood `fernwood-11`). They mean the company is likely multi-brand (README, Research step 2). Each has its own slug, so read its menu the same way, and load every brand the user confirms.

## 2. Read it

The menu is public JSON and needs no auth. LeafLink's robots.txt blocks web-fetch tools, so use curl or Python, or `fetch()` from a browser tab on app.leaflink.com.

```bash
curl -s "https://app.leaflink.com/api/internal/$SLUG/public-menu/brand/"      # name, tagline, website, logo, banner, licenses, sister brands
curl -s "https://app.leaflink.com/api/internal/$SLUG/public-menu/products/"   # {"count","next","results"}
```

Pages hold at most 50 rows. Follow `next` until it is null, and check that the row count equals `count`.

What the brand payload gives you for free:

- **Discovery card and storefront.** `image` is the logo, `banner` is the banner, and `tagline`, `description`, and `website` fill in the rest.
- **State.** Read it from the company name suffix ("Acme - NV") or from `zipcode`/`city`.
- **Licenses.** `company.licenses[].license.number`.
- **Featured product.** `GET /api/internal/$SLUG/public-menu/featured/` returns the brand's own pick. Make it the first `is_featured` product on the storefront.
- **Per-brand storefronts.** Each brand slug has its own brand payload, so each brand's storefront gets its own logo, banner, and description.

## 3. Map each product

| LeafLink | Distru | Rule |
|---|---|---|
| `name` | `name` | Sanitize: `+` → `&`, and drop disallowed characters |
| `sku` | `sku` | As is |
| `description` (HTML) | `description` | Strip tags and keep line breaks |
| `category.name` + `sub_category.name` | `category_id` + `subcategory_id` | See the category table below |
| `strain_names` | `strain_id` | Create the strain once, using the name verbatim. Skip it if empty. |
| `strain_classification` | `group_id` and `strain_type` | See the strain table below |
| `brand.name` | `brand_id` | Your own company, or a created brand company |
| `license.type_code` | THC or hemp | `ADU`/`MED` means THC; `ANC` means hemp or ancillary; null means look for "Hemp"/"CBD" in the name |
| `product_specs` ("10 mg", "72.00 %") | `total_thc` / `total_cbd` + unit | Edibles: the package total in `MG` (e.g. "100mg THC per package" in the description). Flower, vapes, concentrates: `PERCENT`. |
| `unit_multiplier`, "1 case = 12" in the description | `units_per_case` | Prefer the structured field |
| `featured_image`, `images[]` | photos | `featured_image` gets rank 0, extras get 1–3 |
| `id` | `leaflink_product_id` | Lets a later LeafLink connection match the product |
| price | `unit_price` | Public menus hide prices, so send 0 and say so in the receipt |
| `quantity` | (ignore) | Not real inventory |

**Category table** (seeded names; reuse them):

| LeafLink says | Seeded category |
|---|---|
| Flower, packaged 3.5g/7g/1g | Bud/Flower - Each |
| Pre-Rolls (not infused) | Raw Pre-Roll - Each |
| Infused pre-rolls | Infused Pre-Roll - Each |
| Vapes, cartridges, disposables | Vape Cartridge - Each |
| Concentrates | Concentrate/Extract - Each (subcategories already exist) |
| Edibles & Ingestibles (gummies, chocolate, drinks) | Infused Edible - Each |
| Tinctures / Topicals | Tincture - Each / Topical - Each |
| Hemp or CBD-only items | New category "Hemp CBD Edible - Each" (`EDIBLES`) |

A retail-sized item counts as "Each" even when LeafLink sets `sold_in_bulk` (that means sold by the case). Only true bulk (lb, kg) goes to "- Bulk".

**Strain table:**

| `strain_classification` | `strain_type` | Group |
|---|---|---|
| Indica / Sativa / Hybrid | `INDICA` / `SATIVA` / `HYBRID` | Same name |
| Indica Hybrid / Sativa Hybrid | `INDICA_DOMINANT` / `SATIVA_DOMINANT` | Hybrid |
| High CBD, "1:1 CBD", or other ratios | `HIGH_CBD` | High CBD |
| N/A | none | High CBD for hemp CBD items, otherwise none |

When SKUs disagree on a strain's type, use the majority. The field always wins over the prose.

## 4. Create, in this order

1. Rename the account if needed, and create a brand for each extra brand the user confirmed (api §4).
2. Create the categories you need beyond the seeded ones.
3. Create subcategories that add information (Gummies, Beverages). Skip ones that just repeat the category.
4. Create the strains.
5. Create the products with `menu_visibility: "INCLUDE_IN_ALL"`.
6. Upload the photos: send the image URLs to `/api/product-images` (api §5).
7. Mark 4–8 featured products (`is_featured: true`): the LeafLink featured product, plus a bestseller or two per product line. These fill the "Featured Products" row on the storefront. With one storefront per brand, mark 2–4 per brand, starting with each brand's own featured product.

Then go back to the main guide.

## Reference run (a 35-product edibles menu)

- **Products:** 35. 15 THC gummies go to "Infused Edible - Each" with `PACKAGE` tracking. 20 hemp CBD gummies and sparkling waters go to "Hemp CBD Edible - Each" with `PRODUCT` tracking.
- **Subcategories:** Gummies and Beverages.
- **Groups:** Indica 4, Sativa 5, Hybrid 5, High CBD 21.
- **Photos:** 43 on 35 products.
- **Case packs:** 10 for the hemp gummies, 12 for the waters.
