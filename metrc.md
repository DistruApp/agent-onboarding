# Distru onboarding playbook: Metrc inventory

Part of the Distru onboarding guide for AI agents: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md. Optional: run it only when the user gave you a Metrc API key, after the storefront is live.

**Goal:** every active Metrc package lands in Distru as stock, so the storefront shows real availability. That includes products that are not on the menu yet.

**Status:** steps 2 to 6 ran on staging on Oct 30, 2026 (account 81369: 21 active packages, 12,960 units visible in Distru). Step 7, the import itself, has not run yet.

## Why it matters before you create products

Metrc-tracked THC products must use `inventory_tracking_method: "PACKAGE"`, and that setting is permanent. So ask the stock question in stage 3 of the main guide, before the catalog is built. If the user skips Metrc, use `PRODUCT` and never come here.

## 1. Collect two things from the user

1. **Their Metrc user API key.** In Metrc, it's under their name, top right → API Keys.
2. **Which licenses to connect.** Use the license numbers you found in discovery, and confirm them: "I see OCM-DIST-66-975678-DX1. Connect that one?"

## 2. Metrc must be switched on for the account (Distru staff)

Turning Metrc on takes a Distru superadmin. A normal admin gets "Unauthorized". Check first:

```graphql
{ currentUser { company { complianceType activeModules } } }
```

If `complianceType` is `NONE`, tell the user "a Distru specialist is switching on your Metrc connection", keep building the storefront, and check again before step 3. On staging, switching it on also points the account at the Metrc test server.

## 3. Save the keys

Save the company key first. The license in step 4 is refused until it exists. `description` is required on both.

```graphql
mutation($i: CreateApiKeyInput!) { createApiKey(input: $i) { id } }
# company key: {"i": {"type": "METRC", "metrcKey": "<user key>", "description": "Company key"}}
# user key:    {"i": {"type": "METRC", "metrcKey": "<user key>", "userId": "<admin user id>", "description": "User key"}}
```

## 4. Add each license

Get the valid license types for the state from `GET /api/v1/licenses/form_options`, then:

```bash
curl -s -X POST https://staging.distru.com/api/v1/licenses -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' -d '{
  "license": {"company_id": 81369, "license_number": "OCM-DIST-66-975678-DX1", "license_type": "AU Distributor",
              "compliance_type": "METRC", "active": true,
              "issue_date": "2026-01-01T00:00:00Z", "expiry_date": "2027-12-31T00:00:00Z"}}'
# → 200 {"id": 14648, ...}
```

`company_id` is required even though it's your own company. Creating the license starts a background pull of Metrc items and strains.

## 5. Give each license a home

Create a location linked to the license, then make it the intake location:

```bash
POST /public/v1/locations {"name":"WYLD Warehouse","street_address":"...","city":"...","state":"NY","zip":"...","country":"US","license_id":"14648"}
```

```graphql
mutation($i: UpdateLicenseInput!) { updateLicense(input: $i) { id } }
# {"i": {"id": "14648", "intakeLocationId": "<location id>"}}
```

## 6. Pull packages

Ask Distru for the packages. Leave items and strains out: the pull from step 4 is already fetching them, and asking again is refused.

```graphql
mutation($i: MetrcBackfillInput!) { backfillMetrcCaches(input: $i) { __typename } }
# {"i": {"licenseId": "14648", "since": "2026-01-01T00:00:00Z", "includePackages": true, "includeLocations": true, "includeTransfers": true}}
```

Within about a minute, `GET /public/v1/metrc/packages` lists them. Keep rows with `status: "ACTIVE"`. Progress is visible on `license(id) { metrcPullStatus { hasInProgressBackfills cachesUpdatedUpUntil { packages } } }`.

## 7. Map and import (not yet verified)

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
