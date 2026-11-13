# Finishing the setup with the Distru connector

Part of the Distru onboarding guide for AI assistants: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md

After the setup page, the person adds the Distru connector (Claude: Settings → Connectors, which the page's button opens with Distru filled in) and signs in with their new login. Its tools all start with `distru-`. Every tool takes an `intent` (their end goal in one sentence) and a `userSatisfactionEstimate` (5 on track, down to 1 stuck); fill both honestly, because Distru's team reads them to improve onboarding.

Tool reference: https://docs.distru.com/mcp

## §1 Check the connection

1. `distru-get-me` shows who's signed in. The email must be the login from the setup page. If it isn't, they signed in to another Distru account: ask them to disconnect the connector and sign in with the new login.
2. `distru-search-products` (`limit: 1`) for `totalCount`, and `distru-search-menus` for the storefront and its public URL.
3. Post one line: "✓ Connected to your Distru account: Acme Edibles, 35 products, storefront live."

If the Distru tools aren't in your tool list, the connector isn't on for this chat (README, "If something goes wrong").

## §2 Review the catalog

Compare what's in Distru with what you found in research.

- **Count.** Products in Distru should equal the catalog. The page skips products it couldn't create and says why; recreate them (§4).
- **Names, categories, potency.** Spot-check five products with `distru-search-products`. Fix one with `distru-mutate-product` (`UPDATE`), or the same change on many with `distru-bulk-update-products`.
- **Brands.** The page puts every product under the account's own company. For multi-brand companies: create each extra brand with `distru-mutate-company` (`CREATE`, `name`), then move its products with `distru-bulk-update-products` (`ids`, `brandId`).
- **Featured products.** Pick 4 to 8: the source's own featured pick, then one or two bestsellers per product line. `distru-bulk-update-products` with `isFeatured: true`. With one storefront per brand, 2 to 4 per brand. Search rows don't show the flag; `distru-get-product` does.
- **Tracking method.** `inventoryTrackingMethod` on each product is fixed. THC products for a Metrc or BioTrack shop should be `PACKAGE`; hemp, CBD-only, and non-cannabis items `PRODUCT`. If a product is wrong, don't try to change it: tell them, and a Distru specialist will fix it.

## §3 Polish the storefront

The page builds one storefront: a Featured row plus one row per category, prices hidden. If `distru-search-menus` finds none (the page's storefront step failed), create it: `distru-mutate-menu` (`CREATE`, `name`, `locationIds` from `distru-search-locations`, `visibility: PUBLIC`, `active: true`, `hidePricesMode: HIDE_FOR_EVERYONE`, `collections`).

- **Read it:** `distru-get-menu-settings` with `includeCollections: true`. The result is `{menu: {...}}`; the rows are `menu.collections`, each with an `id`.
- **Rows:** `distru-mutate-menu` (`UPDATE`, `collections`). The list replaces the stored one: every row you leave out is deleted. So always start from `menu.collections`, send `{id}` for each row you keep, and add new rows without an `id`. If you read no rows, stop and read again; never send a list with only your new row. Keep "Featured Products" (`onlyFeaturedProducts: true`) first, then 3 to 5 rows by product line (`productCategoryIds`, `productSubcategoryIds`) or brand (`brandIds`). A product joins a row when it matches every filter set; empty rows are hidden.
- **Multi-brand, one storefront:** a row per brand right after Featured.
- **One storefront per brand:** `distru-mutate-menu` (`CREATE`, `name`, `locationIds`, `urlSuffix` in plain lowercase ASCII) for each extra brand, then pin products with `distru-bulk-update-products` (`ids`, `menuVisibility: INCLUDE_IN_SELECT`, `menus`). Logos and banners are set in Distru (Commerce → menu → Appearance); the connector doesn't change appearance.
- **Prices:** once real wholesale prices are in (§4), switch `hidePricesMode` to `HIDE_FOR_NON_LOGGED_IN_USERS`, so retailers see prices after they log in.
- Open the storefront URL yourself and check that each row shows products.

## §4 Products and prices

**Creating products** (spreadsheet, website, pasted list, or ones the page skipped). One `distru-mutate-product` (`CREATE`) per product; the result is `{product: {...}}`:

| Field | Value |
|---|---|
| `name`, `sku` | Unique in the company. No `:`, `[` or `]`. Make stable SKUs from brand and product if they have none (`ACME-MANGO-100`), and say so. |
| `categoryId` | `distru-search-lookups` (type `PRODUCT_CATEGORY`). Reuse the seeded state categories. |
| `unitTypeId` | `distru-search-lookups` (type `UNIT_TYPE`): Unit for packaged goods |
| `vendorId` | Their own company (`distru-search-companies`) |
| `unitPrice` | Their wholesale price, or 0 if unknown |
| `inventoryTrackingMethod` | `PACKAGE` for THC products when they use Metrc or BioTrack; `PRODUCT` otherwise and for hemp, CBD-only, and non-cannabis items. Permanent. |
| `menuVisibility` | `INCLUDE_IN_ALL`, to put it on the storefront |
| `totalThc` / `totalCbd` + `totalCannabinoidUnit` | `MG` package total for edibles and drinks, `PERCENT` for flower, pre-rolls, vapes, concentrates |

The connector can't add photos. Products the page loaded already have theirs; for new ones, say photos can be added in Distru.

**Prices.** When they send a price sheet, match rows to products by SKU, then name. Set `unitPrice` (and `wholesaleUnitPrice` if they give both) with `distru-mutate-product` per product, or `distru-bulk-update-products` for a group at one price. Then switch the storefront's price mode (§3).

## §5 Stock without Metrc

For `PRODUCT`-tracked products only. They need a location, which the page created (`distru-search-locations`).

`distru-adjust-inventory` with `productId`, `quantity` (the count), `reason: OTHER`, and `locationId`. Add `costPerUnit` if they gave costs. One call per product; adjustments can't be undone, so check the counts with them first.

Metrc stock: metrc.md.

## §6 Hand off

1. `distru-search-products` count equals the catalog, across every brand.
2. Every storefront URL loads and shows its rows.
3. Send the receipt (README). The login is the email they gave; the password is the one on their setup page. Never ask them for it.
