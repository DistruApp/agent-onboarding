# LeafLink

Part of the Distru onboarding guide for AI assistants: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md

The setup page reads a LeafLink menu for you, so all you need is the menu URL. Verified end to end on a 35-product edibles menu with 43 photos, and dry-run on flower, pre-roll, and vape menus.

## 1. Find the menu

- **They gave a URL** such as `https://app.leaflink.com/menu/acme/`. Use it as `source` in the link.
- **They gave only a brand name.** Search `site:app.leaflink.com "<brand>"`, or try the brand name lowercased with dashes as the slug: `https://app.leaflink.com/menu/<brand-slug>/`.
- LeafLink blocks most web-fetch tools, so you may not be able to open the menu yourself. That's fine: the page reads it server side and shows the person the product count before they click. If you're unsure the slug is right, say so in the plan card; the page tells them if the link doesn't load.

## 2. Sister brands

A menu covers one brand, so it can hide the others. LeafLink lists sister brands on the menu page ("More from this seller"). If you can read the page (a browser or a shell), the brand payload at `https://app.leaflink.com/api/internal/<slug>/public-menu/brand/` lists them under `company.additional_brands`, each with its own slug. Otherwise rely on their website and ask (README, "Brand setup").

The page loads one menu. For each extra brand they confirm, load its products after they connect (connector.md §4), or ask them for a product export.

## 3. What the page maps

| LeafLink | Distru |
|---|---|
| name, SKU, description | Product name (`+` becomes `&`, disallowed characters dropped), SKU, description |
| category and subcategory | The state's seeded category where one fits, otherwise a new category with the right type |
| strain classification | Product group (Indica, Sativa, Hybrid, High CBD) |
| featured product, images | Featured flag, photos (the first one is the hero) |
| brand logo, banner, description | The storefront's branding |
| price | Not public, so prices start at 0 and stay hidden on the storefront |
| quantity | Ignored. Stock comes from Metrc or their counts. |

After they connect, spot-check potency and strain names (connector.md §2); public menus often leave them out.

## 4. Prices

Public menus hide wholesale prices. Ask for a price sheet in any format, such as a LeafLink product export or their line sheet, and load it with the connector (connector.md §4). Don't ask for their LeafLink API key in the chat.
