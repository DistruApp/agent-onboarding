# Distru onboarding API cookbook

Part of the Distru onboarding guide for AI agents: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md. Every example below ran against staging on Oct 29–30, 2026, unless it is marked *unverified*.

## §0 Conventions

| | |
|---|---|
| App + internal API | `https://staging.distru.com/api/v1/...` and `https://staging.distru.com/graphql` (both use the session JWT) |
| Public API | `https://staging.distru.com/public/v1/...` (uses the API key). Full reference: https://apidocs.distru.dev/llms-full.txt |
| Auth header | `Authorization: Bearer <token>` plus `Content-Type: application/json` |
| Ids | Send every id **as a string** in request bodies. Numbers are rejected. |
| Writes | `POST` creates when there is no `id` and updates when there is one. Updates are sparse. |
| Lists | Return `{"data": [...], "next_page": ...}`. Follow `next_page` exactly until it is null. |
| Errors | Always `{"errors":[{"message","pointer"}]}`. Fix the field that `pointer` names and retry. |

**No shell, only a browser tab?** Run `fetch()` from a tab on `https://staging.distru.com` (a plain JSON page such as `/api/v1/registrations/onboarding_survey_form_options` works). The public API does not send CORS headers, so calls have to come from that origin.

## §1 Create the account

**1a. Get a sign-up id.** Use one of these paths.

**Path A: the user has a link.** It looks like `https://staging.distru.com/sign_up/<sign_up_id>`. Verified.

**Path B: Distru staff with a staging superadmin login.** Unverified.

```bash
curl -s -X POST https://staging.distru.com/api/v1/sessions -H 'Content-Type: application/json' \
  -d '{"session":{"email":"'$ADMIN_EMAIL'","password":"'$ADMIN_PASSWORD'"},"verify_session_limit":false}'
curl -s -X POST https://staging.distru.com/api/v1/registrations -H "Authorization: Bearer $ADMIN_JWT" -H 'Content-Type: application/json' \
  -d '{"company":{"company":{"name":"WYLD","us_state":"NV","email":"owner@wyld.com","category":"Manufacturer","demo":true,"skip_chargeover_subscription":true,"skip_analytics_sync":true}}}'
# → 201 {"id": 81368, "sign_up_id": "c8868077-..."}
```

The double-nested `company` is intentional. `category` must be one of: Dispensary, Delivery, Cultivator, Manufacturer, Distributor, Microbusiness, Lab, Other.

**Path C: the demo provisioning API.** Unverified. It needs the `x-demo-api-key` header, which equals the server's `DEMO_ADMIN_API_KEY` (ask engineering). One call creates the org, the company, and an admin, and turns on the API and Distru Commerce modules. Sibling endpoints: `/demo/api-key`, `/demo/location`, `/demo/license`, `/demo/menu`. They only work on demo companies.

```bash
curl -s -X POST https://staging.distru.com/demo/account -H "x-demo-api-key: $DEMO_KEY" -H 'Content-Type: application/json' \
  -d '{"company":{"name":"WYLD","us_state":"NV","organization_name":"WYLD","category":"Manufacturer"},"user_email":"owner@wyld.com"}'
# → {company_id, organization_id, user_id, password}
```

**1b. Create the first user (Path A or B).** No auth is needed. The sign-up id works once.

1. Read the survey choices with `GET /api/v1/registrations/onboarding_survey_form_options`.
2. Create the user:

```bash
curl -s -X POST https://staging.distru.com/api/v1/registrations/user -H 'Content-Type: application/json' -d '{
  "user": {"sign_up_id": "<id>", "company_id": 81368, "email": "owner@wyld.com",
           "password": "'$TEMP_PASSWORD'", "password_confirmation": "'$TEMP_PASSWORD'",
           "terms_of_service_checked": true, "profile": {"first_name": "Ada", "last_name": "Owner", "title": ""}},
  "onboarding_survey": {"company_roles": ["Brand (owns your own brand)"], "next_features": [], "user_departments": ["Executive"], "user_roles": ["Executive"]}}'
# → 201 {"jwt", "refresh_token", "user"}   you are now logged in as the admin
```

## §2 Keys and session

- **Session JWT.** It comes from §1b or from `POST /api/v1/sessions {"session":{"email","password"},"verify_session_limit":false}`. Use it for `/graphql` and `/api/v1/*`.
- **API key.** Fetch it once with the JWT and keep it in a variable. Never print it.

```bash
curl -s -X POST https://staging.distru.com/graphql -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"query":"{ myDefaultMcpApiKeyPlaintext }"}'     # → "distru_mcp_..."; works on /public/v1 immediately
```

- **Release the session last, after the storefront is done.** Distru allows one session per user.

```bash
curl -s -X POST https://staging.distru.com/api/v1/sessions/delete -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"tokens":["'$JWT'","'$REFRESH'"]}'              # → {"ok": true}
```

If a key ever leaks, rotate it with `mutation { regenerateDefaultMcpApiKey { id } }`.

## §3 Read the defaults

Read these before you create anything: `GET /product-categories`, `/product-subcategories`, `/product-groups`, `/unit-types`, `/strains`, `/locations`, `/companies`.

A new account already has:

| Object | Seeded |
|---|---|
| Categories | The state's compliance categories (21 for NY: "Bud/Flower - Each", "Raw Pre-Roll - Each", "Infused Edible - Each", "Vape Cartridge - Each", ...). Some states get none. |
| Subcategories | Concentrates only |
| Product groups | Indica, Sativa, Hybrid, High CBD |
| Unit types | Active: Pound, Gram, Ounce, Liter, Milliliter, Unit |
| Companies | One row: the account's own company. Use it as `vendor_id`. Its `relationship_type` is null. |
| Not created | Locations, licenses, strains, price tiers, products, inventory |

## §4 Taxonomy

| Need | Call |
|---|---|
| Rename the account / set the website | `POST /companies {"id":"<self id>","name":"WYLD","website":"https://..."}`. This renames the whole account. |
| Another brand | `POST /companies {"name":"Good Tide","category":"Manufacturer"}` → use the id as `brand_id` |
| New category | `POST /product-categories {"name":"Hemp CBD Edible - Each","official_product_category_id":"EDIBLES"}` |
| Subcategory | `POST /product-subcategories {"name":"Gummies","product_category_id":"51104"}` |
| Strain | `POST /strains {"name":"GMO Zkittles","strain_type":"INDICA_DOMINANT"}` |

- `official_product_category_id` is one of: `FLOWER`, `PREROLLS`, `VAPES`, `CONCENTRATES`, `EDIBLES`, `TINCTURES`, `TOPICALS`, `CLONES`.
- `strain_type` is one of: `INDICA`, `SATIVA`, `HYBRID`, `INDICA_DOMINANT`, `SATIVA_DOMINANT`, `HIGH_CBD`.

## §5 Products, photos, and stock

**Product.** Send one `POST /products` per SKU. It returns 200.

```json
{"name":"Elderberry 2:1 CBN & Indica Enhanced Gummies | 100:50mg","sku":"FG-110108-NVV1",
 "description":"The bestselling infused gummy in the U.S. ...",
 "category_id":"51104","subcategory_id":"64723","group_id":"6654","strain_id":null,
 "vendor_id":"69456","brand_id":"69456","unit_type_id":"58665","unit_price":0,
 "inventory_tracking_method":"PACKAGE","total_cannabinoid_unit":"MG","total_thc":"100",
 "units_per_case":null,"leaflink_product_id":"440133","menu_visibility":"INCLUDE_IN_ALL"}
```

Field rules:

- **Name.** Allowed characters are letters, numbers, spaces, and `: [ ] ~ # - $ / | % & ' ( ) .` only. Replace `+` with `&` and drop everything else.
- **Tracking method.** `inventory_tracking_method` can never change later. Use `PACKAGE` for Metrc-tracked THC products and `PRODUCT` for everything else.
- **Cannabinoids.** Use `MG` for edibles and drinks, as the package total. Use `PERCENT` for flower, pre-rolls, vapes, and concentrates. Whenever you send THC or CBD, also send `total_cannabinoid_unit`.
- **Net weight.** For flower, pre-rolls, vapes, and concentrates, set `unit_net_weight` and `unit_net_weight_and_serving_size_unit_type_id` (the Gram id).
- **Menu visibility.** `menu_visibility: "INCLUDE_IN_ALL"` puts the product on every storefront menu.

**Photos.** `POST /product-images` as multipart with fields `file`, `product_id`, and `rank` (0–3; 0 is the hero image). Each file can be up to 10 MB. It returns 201.

```bash
curl -s -o img.jpg "$IMAGE_URL"
curl -s -X POST "$BASE/product-images" -H "Authorization: Bearer $KEY" -F file=@img.jpg -F product_id=$PID -F rank=0
```

Getting photos across with only a browser: the browser can't download images across origins (no CORS), but `window.name` survives navigation within a tab.

1. Open a tab on the image host (e.g. the cloudfront URL of one image).
2. `fetch()` every image from that same origin. Resize to 1600px JPEG on a canvas.
3. Store them as data URLs in `window.name`.
4. Navigate the same tab to a staging.distru.com JSON page.
5. Read `window.name` there and upload with `FormData`.

This moved 43 photos (9 MB) in one hop.

**Stock without Metrc (PRODUCT tracking only).** It needs a location first:

`POST /adjustments {"product_id","location_id","quantity":100,"reason":"other","unit_cost":12,"completion_datetime":"2026-10-30T00:00:00Z"}`

## §6 Storefront (Distru Commerce menu)

All verified, except that **the link only resolves once the company's DISTRU_COMMERCE module is on.** That takes a superadmin today, or Path C, which turns it on at creation. Until then the page says "This Menu Can't Be Found".

1. **Location.** Required by the menu. The address is geocoded, so it must be real.
   `POST /locations {"name":"WYLD Warehouse","street_address":"...","city":"...","state":"NV","zip":"...","country":"US"}`
2. **Products on the menu.** Set `menu_visibility: "INCLUDE_IN_ALL"` on each product.
3. **Featured products.** Pick 4 to 8 hero SKUs and set `is_featured: true` on each with `POST /products {"id","is_featured":true}`. Start with the source's own featured product (LeafLink: `/public-menu/featured/`). Then add one or two per product line, favoring anything described as a bestseller.
4. **Logo and banner.** Upload with the JWT: `POST /api/v1/images` multipart with field `image`. It returns `{"id"}`.
5. **Menu with collections.** GraphQL with the JWT. Add `primaryUrl` to an `upsertMenu` call when you change anything later.

**Collections decide what the homepage shows.** Official collections (one per state category, plus All Products) only render as small category tiles. The product rows on the homepage come from custom collections. Without them, a buyer sees the banner, two tiles, and nothing else.

Each custom collection is a saved filter:
- A product joins the collection when it matches every filter you set.
- Within one filter, matching any of the values is enough.
- Empty collections are hidden.

Plan 3 to 5 rows:
- "Featured Products" (`featured: true`) always goes first.
- Then add one row per subcategory or product line (`productSubcategoryIds`, `productCategoryIds`), brand (`brandIds`), or effect (`productGroupIds`).

```graphql
mutation($input: CreateMenuInput!) { createMenu(input: $input) { id primaryUrl } }
# input:
{"name":"WYLD Wholesale Menu","externalName":"WYLD Wholesale","urlSuffix":"wyld-wholesale",
 "locationIds":["23366"],"active":true,"visibility":"PUBLIC",
 "showProductsWithNoInventoryMode":"SHOW_AND_ALLOW_TO_ORDER","allowCustomersToOverOrder":true,
 "hidePricesMode":"HIDE_FOR_NON_LOGGED_IN_USERS","sellByCaseOnly":false,
 "menuCollections":[
   {"name":"Featured Products","featured":true},
   {"name":"THC Gummies","productCategoryIds":["51104"]},
   {"name":"Hemp CBD Gummies","productSubcategoryIds":["64724"]},
   {"name":"CBD Sparkling Waters","productSubcategoryIds":["64725"]}]}
```

`menuCollections` replaces the whole custom list on every `upsertMenu`, so always resend it. Other filter fields you can use: `brandIds`, `productGroupIds`, `strainIds`, `strainTypes`, `officialProductCategoryIds`, and `thcRangeMin`/`thcRangeMax`/`thcRangeUnit`.

6. **Branding.**

```graphql
mutation($i: UpsertMenuInput!) { upsertMenu(input: $i) { id } }
# i = the same fields as create (minus urlSuffix, and including menuCollections) + "id", "primaryUrl", and:
{"accentColor":"#B4532A","companyLogoId":"19694","showCompanyLogo":true,
 "bannerImages":[{"imageId":"19695","position":0}],"showBannerImage":true,
 "companyDescription":"...","showCompanyDescription":true,"companyInfoUrl":"https://...","showCompanyInfo":true}
```

Menu notes:

- **Selling with no inventory.** `SHOW_AND_ALLOW_TO_ORDER` first needs `mutation { updateCompanySettings(input:{id:"<companyId>", allowNegativeAvailableQuantity:true}) { id } }`.
- **Hiding prices.** `hidePricesMode` is one of `DO_NOT_HIDE`, `HIDE_FOR_NON_LOGGED_IN_USERS`, or `HIDE_FOR_EVERYONE`. Hide prices while they are $0.
- **Link format.** `https://staging.distru.com/menu/<company-slug>/<urlSuffix>`. `GET /public/v1/menus` returns `url` and `product_count`.

## §7 Gotchas we hit

- Numeric ids are rejected. Send strings.
- A category without `official_product_category_id` fails with "Select a category type".
- Product names reject `+` and commas.
- `upsertMenu` is not sparse. Resend the menu's core fields, `locationIds`, `primaryUrl`, and `menuCollections`.
- A menu with no custom collections looks empty: only category tiles, no product rows.
- `showTotalCbd` errors unless selling by potency is on.
- One session per user. Release yours, or the user sees "Login Limit Reached".
- `toggleComplianceIntegration` (turning on Metrc) and module activation return Unauthorized for a normal admin. Hand them to Distru staff.
