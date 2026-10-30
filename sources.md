# Distru onboarding playbook: Shopify, spreadsheets, Apex Trading, websites, and starting from nothing

Part of the Distru onboarding guide for AI agents: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md. Map every source onto the same Distru fields as the LeafLink playbook ([leaflink.md](https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/leaflink.md), section 3), and create records in the same order.

## Shopify store (verified read)

Signs of Shopify: images on `cdn.shopify.com`, or `/products/` and `/collections/` pages. Use the store's home URL.

- **Read it in the browser:** open a second tab on the store (not the Distru tab) and `fetch()` the JSON there.
- **Products:** `/products.json?limit=250` gives names, descriptions (HTML), product types, tags, variants with SKUs, and image URLs. Follow `&page=2` and on until a page comes back empty.
- **Categories:** `/collections.json`, then `/collections/<handle>/products.json` for each. Skip catch-all, sale, and state collections ("All", "New", "Best sellers", "Nevada"). Put each product in its largest remaining collection.
- **Photos:** add `&width=1200` to `cdn.shopify.com` image URLs before you send them to `/api/product-images`.
- **Prices:** retail sites show retail prices, not wholesale. Send 0 unless they confirm the prices are wholesale.
- **Faster option:** from the Distru tab, `onboard('/import', {action: 'plan', source: 'https://<store>'})` reads it for you and returns the same mapped shape as a LeafLink menu (leaflink.md §2).

## Spreadsheet, CSV, or Google Sheet (mapping verified)

1. Read the file. Find the header row, which isn't always row 1, and skip blank and total rows.
2. Match columns by meaning, not exact header text:

| Distru field | Typical headers |
|---|---|
| `name` | Product, Product Name, Item, Description |
| `sku` | SKU, Item #, Product Code, UPC |
| category | Category, Type, Product Type, Class |
| subcategory | Subcategory, Sub Type, Form |
| strain / group | Strain, Cultivar, Genetics, Indica/Sativa/Hybrid |
| `unit_price` | Wholesale, Price, Unit Price, Case Price ÷ units per case |
| `units_per_case` | Case Qty, Units/Case, Pack |
| potency | THC, THC %, THC mg, CBD |
| brand | Brand, Line |
| photo | Image, Image URL, Photo |
| stock (optional) | Qty, On Hand, Available |

3. Show the mapping to the user in one line and keep going unless they object. For example: "I'll read Product as name, Item # as SKU, Wholesale as price, and skip the Notes column."
4. If there is no SKU column, generate stable SKUs from the brand and product (e.g. `ACME-MANGO-100`) and say so in the receipt.
5. Prices from a spreadsheet are real, so load them. This is the main advantage over a public menu.
6. Stock counts load as stock adjustments (API cookbook §5), but only when the user does not use Metrc. With Metrc, inventory comes from Metrc.

## Apex Trading (not yet verified)

- Apex brand storefronts are public. Search "<brand> Apex Trading" and read the page.
- If you can't get structured product data in one or two tries, ask: "Can you export your Apex products as a CSV? (Products → Export)". Then follow the spreadsheet flow.

## Brand website only (best effort)

- Product pages usually give name, category, strain, potency, description, and photo. Prices and SKUs are usually missing.
- Build the list, then confirm it in one message before you import: "I found 18 products on acme.example. Import all 18?"

## Nothing yet

Ask once: "Paste your product list in any format, even a photo of a price sheet, and I'll take it from there."

- If they send a photo, read it and turn it into rows. Confirm the row count.
- If they have nothing, still create the account and the branded storefront with what you found (logo, banner, description, address). Say products can come in anytime: they paste a list and you load it.
