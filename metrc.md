# Distru onboarding playbook: Metrc inventory

Part of the Distru onboarding guide for AI agents: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md. Run it after the catalog is in.

**Goal:** every active Metrc package lands in Distru as inventory, including products that are not on the menu.

**Status:** the staging mock is set up and verified. Steps 2 to 6 come from the code and have not run yet, because step 2 needs Distru staff.

## Why it matters before you create products

Metrc-tracked THC products must use `inventory_tracking_method: "PACKAGE"`, and that setting is permanent. So decide on Metrc in stage 3 of the main guide, before the catalog is built.

## 1. Collect two things from the user

1. **Their Metrc user API key.** In Metrc, it's under their name, top right → API Keys.
2. **Which licenses to connect.** Use the license numbers you found in discovery, and confirm them: "I see OCM-DIST-66-975678-DX1. Connect that one?"

## 2. Enable Metrc on the account (Distru staff, once per account)

A normal admin gets "Unauthorized" for these, so tell the user "a Distru specialist is flipping on your Metrc connection" and continue with what you can do. Staff steps:

1. Impersonate the account. In Settings → Metrc, turn on "Enable Metrc Integration". This calls GraphQL `toggleComplianceIntegration(complianceType: METRC)`. On staging, this also points the account at the Metrc mock (`metrc_backend = mock`).
2. Under "Add License", enter each license number exactly as Metrc has it.

## 3. Save the keys (admin)

```graphql
mutation { createApiKey(input: {type: METRC, metrcKey: "<user key>", description: "Company key"}) { id } }
mutation { createApiKey(input: {type: METRC, metrcKey: "<user key>", userId: "<admin user id>"}) { id } }
```

Only do this after step 2. Before step 2, the key would be checked against live Metrc.

## 4. Give each license a home (admin)

Create a location, or reuse one, and link it to the license:

`POST /public/v1/locations {"name":"WYLD Warehouse", ..., "license_id":"<license id>"}`

Then set it as the intake location in Settings → Metrc.

## 5. Pull packages

1. List what Metrc has with `GET /public/v1/metrc/packages`, filtered to active packages on this license.
2. If the list is empty, the cache may only hold changes from the last 24 hours. Ask staff to run "Initiate Backfill" (Settings → Metrc, or `backfillMetrcCaches` with packages included).

## 6. Map and import

1. **Group packages by Metrc item name.**
2. **Exact name match to an existing product.** Use it. Catalog names were built to match.
3. **No match: create the product** (`PACKAGE` tracking), with these rules:
   - **Category:** map the Metrc item category to the seeded one ("Edible (weight - each)" → "Infused Edible - Each").
   - **Unit:** the Metrc unit "Each" maps to Unit.
   - **Brand:** use the item's name prefix when it matches a sister brand from discovery (Good Tide, Grön).
   - **Tell the user:** this is the magic moment. "6 products in your Metrc weren't on your Wyld menu. They look like Good Tide and Grön, so I added them under those brands."
4. **Import** with `POST /public/v1/packages/import`:

```json
{"packages":[{"compliance_label":"1A4060300000C3C000000008","product_id":"<uuid>","location_id":"<id>"}]}
```

Up to 300 packages per call, one license per call. The call is all or nothing, so fix the failing row and resend.

5. **Report** the packages imported, the total units, and the products created from Metrc.

## Staging mock (Distru staff demos)

- **Site.** metrc-mock.distru.com mimics Metrc. You can create items and packages in its UI. The package form needs an ingredient package, so use a seeded one.
- **Credentials.** Staff test accounts and keys are in the Notion tracker (GTM Agent Hub → Agent Onboarding – Staging Accounts). Never put them in chat.
- **Demo setup.** The WYLD demo mock holds 21 active packages: the 15 Wyld menu gummies plus 4 Good Tide and 2 Grön products that are not on the Wyld menu. That makes the import discover products automatically.
