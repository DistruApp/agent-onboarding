# Spreadsheets, Shopify, Apex Trading, websites, and starting from nothing

Part of the Distru onboarding guide for AI assistants: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md

The setup page loads LeafLink menus and Shopify stores by itself. Everything else, you load after they connect, with the connector (connector.md §4). Either way, send the link first: the account and storefront don't wait for the catalog.

## Shopify store (verified read)

Signs of Shopify: images on `cdn.shopify.com`, or `/products/` and `/collections/` pages. Put the store's home URL in the link as `source`; the page reads `/products.json` and `/collections.json`, maps categories from the largest collection, and loads photos. Retail sites show retail prices, so prices start at 0 unless they confirm they're wholesale.

## Spreadsheet, CSV, or Google Sheet (mapping verified)

1. Read the file. Find the header row, which isn't always row 1, and skip blank and total rows.
2. Match columns by meaning, not exact header text:

| Distru field | Typical headers |
|---|---|
| name | Product, Product Name, Item, Description |
| SKU | SKU, Item #, Product Code, UPC |
| category | Category, Type, Product Type, Class |
| subcategory | Subcategory, Sub Type, Form |
| strain / group | Strain, Cultivar, Genetics, Indica/Sativa/Hybrid |
| price | Wholesale, Price, Unit Price, Case Price ÷ units per case |
| units per case | Case Qty, Units/Case, Pack |
| potency | THC, THC %, THC mg, CBD |
| brand | Brand, Line |
| stock (optional) | Qty, On Hand, Available |

3. Show the mapping in one line and keep going unless they object: "I'll read Product as name, Item # as SKU, Wholesale as price, and skip the Notes column."
4. No SKU column: generate stable SKUs from brand and product (`ACME-MANGO-100`) and say so in the receipt.
5. Prices from a spreadsheet are real, so load them, and switch the storefront to show prices to logged-in retailers (connector.md §3).
6. Stock counts load as adjustments (connector.md §5), only when they don't use Metrc.

## Apex Trading (not yet verified)

Apex brand storefronts are public: search "<brand> Apex Trading" and read the page if you can. If you can't get a clean product list in one or two tries, ask: "Can you export your Apex products as a CSV? (Products → Export)". Then follow the spreadsheet flow.

## Website only (best effort)

Product pages usually give name, category, strain, potency, description, and photo; prices and SKUs are usually missing. Build the list, then confirm it in one message before you create anything: "I found 18 products on acme.example. Add all 18?"

## Nothing yet

Ask once: "Paste your product list in any format, even a photo of a price sheet, and I'll take it from there." If they send a photo, read it into rows and confirm the count. If they have nothing, the account and branded storefront are still ready; products can come in anytime.
