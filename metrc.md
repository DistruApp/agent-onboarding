# Distru onboarding playbook: Metrc inventory

Part of the Distru onboarding guide for AI agents: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md. Run it when the user gives you a Metrc API key, after the storefront is live. Metrc is the source of truth for stock: when you have the key, stock comes from here, not from a menu or a spreadsheet.

**Goal:** every active Metrc package lands in Distru as stock, so the storefront shows real availability. That includes products that are not on the menu yet.

**Status:** verified end to end on Oct 30, 2026. One `/api/metrc` call connected the license, packages appeared about 4 minutes later, and the import brought in 21 active packages (12,960 units) as stock, including 6 products from sister brands that weren't on the menu.

## Why it matters before you create products

Metrc-tracked THC products must use `inventory_tracking_method: "PACKAGE"`, and that setting is permanent. So ask the inventory follow-up early (main guide), before the catalog is built. If the user skips Metrc, use `PRODUCT` and never come here.

## 1. Collect two things from the user

1. **Their Metrc user API key.** In Metrc, it's under their name, top right → API Keys. Never repeat it back.
2. **Which licenses to connect.** Use the license numbers you found in discovery, and confirm them: "I see OCM-DIST-12-345678-AB1. Connect that one?"

## 2. Connect Metrc

One call to the onboarding API switches Metrc on, saves the key, adds each license, makes your storefront location its home, and starts pulling packages.

```bash
curl -s -X POST https://distru-agent-onboarding-api.vercel.app/api/metrc -H 'Content-Type: application/json' -d '{
  "account_token": "'$ACCOUNT_TOKEN'", "metrc_key": "'$METRC_KEY'",
  "licenses": ["OCM-DIST-12-345678-AB1"], "location_id": "<storefront location id>"}'
# → {"metrc": "connected", "licenses": [{"license_id": "14648", "number": "...", "location_id": "..."}], "warnings": []}
```

Field notes:

- **`licenses`.** Each entry is a license number, or `{"number", "type"}` if you know the state's license type.
- **Location.** Send `location_id` to reuse the storefront location. To create a new one instead, send `location`: `{"name","street_address","city","state","zip"}`.
- **`since`.** Optional. How far back to pull. The default is one year.
- **`warnings`.** Read them. They name anything that didn't connect, such as a license with no location.

## 3. Wait for packages

Packages usually show up in `GET /public/v1/metrc/packages` within 2 to 6 minutes; a new license pulls its Metrc history first. Keep the storefront moving while you wait, and poll every 30 seconds. Keep rows with `status: "ACTIVE"` (the list also holds discontinued packages). If the list is still empty after 10 minutes, check that the license numbers are right and that the key belongs to a user with access to them.

Each package row has `label`, `status`, `quantity`, `unit_type`, and `item` (`name`, `category_name`, `strain`).

## 4. Map and import

1. **Group packages by Metrc item name.**
2. **Exact name match to an existing product.** Use it. Catalog names were built to match.
3. **No match: create the product** (`PACKAGE` tracking), with these rules:
   - **Category:** map the Metrc item category to the seeded one ("Edible (weight - each)" → "Infused Edible - Each").
   - **Tracking:** `inventory_tracking_method: "PACKAGE"`.
   - **Unit:** the Metrc unit "Each" maps to Unit.
   - **Brand:** use the item's name prefix when it matches a sister brand from discovery (Bluebird, Fernwood).
   - **Tell the user:** this is the magic moment. "6 products in your Metrc weren't on your menu. They look like Bluebird and Fernwood, so I added them under those brands."
4. **Import** with `POST /public/v1/packages/import`:

```json
{"packages":[{"compliance_label":"1A4000000000000000000001","product_id":"<uuid>","location_id":"<id>"}]}
```

Up to 300 packages per call, one license per call. The call is all or nothing, so fix the failing row and resend. It returns the imported packages with `quantity_available`; add those up for the receipt.

5. **Report** the packages imported, the total units, and the products created from Metrc.

## Reference

The onboarding API makes the Metrc calls for you, so you rarely need Metrc's own API. If you do, each state has its own API host with docs, for example https://api-ca.metrc.com/Documentation (swap `ca` for the state's two-letter code).
