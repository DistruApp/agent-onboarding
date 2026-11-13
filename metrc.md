# Metrc

Part of the Distru onboarding guide for AI assistants: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md

For Metrc shops, Metrc is the source of truth for stock. The goal: every active Metrc package lands in Distru as stock, so the storefront shows real availability, including products that aren't on their menu yet.

## Before the link

- **Ask early.** The inventory answer sets the tracking method for every THC product, and it's permanent. With `inventory=Metrc` in the link, the page creates THC products as package-tracked, which Metrc stock needs.
- **Licenses.** Put the license numbers you found in the link as `licenses` (comma separated), so the page prefills them.

## On the setup page

When the account is ready, Metrc shops see a "Connect Metrc" step. They enter their Metrc user API key (in Metrc: their name, top right → API Keys) and confirm the license numbers. The key goes from the page to Distru and never passes through the chat. Never ask for it in the chat; if they paste it anyway, don't repeat it, and point them to the page.

The page switches Metrc on, saves the key, adds each license with the warehouse as its home, and starts pulling packages. If they skipped it, they can reopen the same setup link: it picks up where it stopped and shows the Metrc step again.

## After they connect the connector

1. **Find the license:** `distru-search-licenses`. Packages appear 2 to 6 minutes after the page connects Metrc, because a new license pulls its history first. Keep polishing the storefront while you wait, and check every minute or so.
2. **List what can come in:** `distru-search-metrc-packages` with `licenseId` and `importStatus: IMPORTABLE`. Follow `nextCursor` to the end. Each row has the label, quantity, unit, and the Metrc item (name, category, strain).
3. **Match each package to a product.** Group by Metrc item name. Use `productMostRecentlyMappedToMetrcItem` when it's set, otherwise an exact or near name match to a catalog product.
4. **No match: create the product** (connector.md §4) with `inventoryTrackingMethod: PACKAGE`, the seeded category that matches the Metrc item category, and Unit for "Each" items. Use the brand when the item name starts with one of their brands. This is the magic moment, so tell them: "6 products in your Metrc weren't on your menu. They look like Bluebird and Fernwood, so I added them under those brands."
5. **Import:** `distru-import-metrc-packages` with `metrcPackageLabel` and `productId` for each package. Up to 300 per call, one license per call, all or nothing: if it fails, fix the row it names and resend.
6. **Report** packages imported, total units, and products created from Metrc.

If nothing shows up after 10 minutes, the license number or the key's access is usually wrong: ask them to check both on the setup page. A Distru specialist can also connect Metrc with them.

## BioTrack and other state systems

Products are created package-tracked from the link (`inventory=BioTrack`), but the page doesn't connect BioTrack. Say a Distru specialist will connect it, and put it in the receipt.
