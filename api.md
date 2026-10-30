# Distru onboarding API cookbook

Part of the Distru onboarding guide for AI agents: https://raw.githubusercontent.com/DistruApp/agent-onboarding/main/README.md. Every example below ran against staging on Oct 29–30, 2026, unless it is marked *unverified*.

## §0 Conventions

You use two APIs.

| API | Base URL | Auth | Use it for |
|---|---|---|---|
| Onboarding API | `https://distru-agent-onboarding-api.vercel.app/api` | None to start. Then the `account_token` or `api_key` it gives you | Creating the account, photos from URLs, the storefront, and connecting Metrc |
| Distru public API | The `public_api` URL from the account response (`https://staging.distru.com/public/v1` on staging) | `Authorization: Bearer <api_key>` | Everything in the catalog: categories, strains, products, locations, stock, Metrc packages. Full reference: https://apidocs.distru.dev/llms-full.txt |

Rules for both:

| | |
|---|---|
| Body | JSON, with `Content-Type: application/json` |
| Ids | Send every id **as a string** in request bodies. Numbers are rejected. |
| Errors | The onboarding API returns `{"error": "..."}`. The public API returns `{"errors":[{"message","pointer"}]}`. Fix what the message names and retry. |

Public API rules:

- `POST` creates when there is no `id` and updates when there is one. Updates are sparse.
- Lists return `{"data": [...], "next_page": ...}`. Follow `next_page` exactly until it is null.

**Staging or production.** Accounts are made on staging unless your prompt says production and gives you an access key. Then add `"environment": "production"` and `"access_key"` to `/api/accounts`, and `"environment": "production"` to `/api/product-images`. The account token carries the environment for every other onboarding call. Take every Distru URL from the account response (`public_api`, `login.url`); the staging hosts below are examples.

**No shell, only a browser tab?** The onboarding API allows calls from any page. The public API does not send CORS headers, so run its calls with `fetch()` from a tab on the account's Distru host (`login.url`; any page there works).

## §1 Create the account

One call creates the company, the first login, and the API key. Distru Commerce is already on.

```bash
ONBOARD=https://distru-agent-onboarding-api.vercel.app/api
curl -s -X POST $ONBOARD/accounts -H 'Content-Type: application/json' -d '{
  "company_name": "Acme Edibles", "us_state": "NV", "email": "ada@acme.example", "phone": "+1 555 010 0100",
  "first_name": "Ada", "last_name": "Owner", "category": "Manufacturer",
  "menu_source": "LeafLink", "source_url": "https://app.leaflink.com/menu/acme/",
  "profile": {"what_they_do": "Own brands and co-packing", "operations": "Licensed in NV; warehouse and kitchen in Las Vegas",
              "inventory_system": "Metrc", "brands": ["Acme", "Bluebird"]}}'
```

On production, add `"environment": "production", "access_key": "<from your prompt>"`.

It returns 201:

```json
{"environment": "staging", "account_id": "12345", "account_token": "acct_...", "api_key": "distru_mcp_...",
 "login": {"url": "https://staging.distru.com", "email": "ada@acme.example", "temporary_password": "..."},
 "public_api": "https://staging.distru.com/public/v1", "modules": ["CORE", "DISTRU_COMMERCE", "..."]}
```

Field notes:

- **`category`.** One of Dispensary, Delivery, Cultivator, Manufacturer, Distributor, Microbusiness, Lab, Other. The default is Manufacturer. Pick it from their answer about what they do.
- **`phone` and `profile`.** Always send them. `profile` holds the user's own answers: what they do, where they operate, their inventory system, and their brands. Distru's team uses them to follow up.
- **`menu_source`.** Optional. One of LeafLink, Apex, Spreadsheet, Website, No menu, Other. Send it with `source_url` so Distru can follow up.
- **One call per company.** Every call creates a new account. If a call fails, read the error before you retry, so you don't create a duplicate.

## §2 Keys

You get two values. Keep both in variables and never print them.

| Value | What it is | Send it to |
|---|---|---|
| `api_key` | The account's key for the public API | The public API (at `public_api`) and `/api/product-images`, as `Authorization: Bearer <api_key>` |
| `account_token` | Proof you created this account. It lasts 30 days | `/api/storefront` and `/api/metrc`, in the body |

The `temporary_password` is the user's. It goes in the receipt and nowhere else.

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
| Rename the account / set the website | `POST /companies {"id":"<self id>","name":"Acme Edibles","website":"https://..."}`. This renames the whole account. |
| Another brand | `POST /companies {"name":"Bluebird","category":"Manufacturer"}` → use the id as `brand_id` |
| New category | `POST /product-categories {"name":"Hemp CBD Edible - Each","official_product_category_id":"EDIBLES"}` |
| Subcategory | `POST /product-subcategories {"name":"Gummies","product_category_id":"51104"}` |
| Strain | `POST /strains {"name":"GMO Zkittles","strain_type":"INDICA_DOMINANT"}` |

- `official_product_category_id` is one of: `FLOWER`, `PREROLLS`, `VAPES`, `CONCENTRATES`, `EDIBLES`, `TINCTURES`, `TOPICALS`, `CLONES`.
- `strain_type` is one of: `INDICA`, `SATIVA`, `HYBRID`, `INDICA_DOMINANT`, `SATIVA_DOMINANT`, `HIGH_CBD`.

## §5 Products, photos, and stock

**Product.** Send one `POST /products` per SKU. It returns 200.

```json
{"name":"Mango Haze Gummies | 100mg","sku":"ACME-MANGO-100",
 "description":"Small-batch fruit gummies ...",
 "category_id":"51104","subcategory_id":"64723","group_id":"6654","strain_id":null,
 "vendor_id":"50001","brand_id":"50001","unit_type_id":"58665","unit_price":0,
 "inventory_tracking_method":"PACKAGE","total_cannabinoid_unit":"MG","total_thc":"100",
 "units_per_case":null,"leaflink_product_id":"123456","menu_visibility":"INCLUDE_IN_ALL"}
```

Field rules:

- **Name.** Allowed characters are letters, numbers, spaces, and `: [ ] ~ # - $ / | % & ' ( ) .` only. Replace `+` with `&` and drop everything else.
- **Tracking method.** `inventory_tracking_method` can never change later. Use `PACKAGE` for Metrc-tracked THC products and `PRODUCT` for everything else.
- **Cannabinoids.** Use `MG` for edibles and drinks, as the package total. Use `PERCENT` for flower, pre-rolls, vapes, and concentrates. Whenever you send THC or CBD, also send `total_cannabinoid_unit`.
- **Net weight.** For flower, pre-rolls, vapes, and concentrates, set `unit_net_weight` and `unit_net_weight_and_serving_size_unit_type_id` (the Gram id).
- **Menu visibility.** `menu_visibility: "INCLUDE_IN_ALL"` puts the product on the storefront.

**Photos from URLs.** Send up to 10 per call to the onboarding API. It downloads each image and attaches it, so you never handle the files. `rank` is 0–3, and 0 is the hero image. Each image can be up to 10 MB.

```bash
curl -s -X POST $ONBOARD/product-images -H "Authorization: Bearer $API_KEY" -H 'Content-Type: application/json' -d '{
  "images": [{"product_id": "1234", "image_url": "https://images.example.com/mango.jpg", "rank": 0},
             {"product_id": "1234", "image_url": "https://images.example.com/mango-back.jpg", "rank": 1}]}'
# → {"uploaded": 2, "failed": 0, "results": [...]}
```

On production, add `"environment": "production"` to the body. Retry only the rows marked `"ok": false`.

**Photos the user sent you as files.** Upload them to the public API as multipart, with fields `file`, `product_id`, and `rank`:

```bash
curl -s -X POST $PUBLIC_API/product-images -H "Authorization: Bearer $API_KEY" \
  -F file=@photo.jpg -F product_id=1234 -F rank=0
```

**Stock without Metrc (PRODUCT tracking only).** It needs a location (§6 step 1) first:

`POST /adjustments {"product_id","location_id","quantity":100,"reason":"other","unit_cost":12,"completion_datetime":"2026-10-30T00:00:00Z"}`

## §6 Storefront (Distru Commerce)

1. **Location.** The storefront ships from a location. The address is geocoded, so it must be real. Use the brand's warehouse or business address from discovery.
   `POST /locations {"name":"Acme Warehouse","street_address":"...","city":"...","state":"NV","zip":"...","country":"US"}`
2. **Products on the storefront.** Set `menu_visibility: "INCLUDE_IN_ALL"` on each product. For one storefront per brand, see "Multi-brand" below.
3. **Featured products.** Pick 4 to 8 hero SKUs and set `is_featured: true` on each with `POST /products {"id","is_featured":true}`. Start with the source's own featured product (LeafLink: `/public-menu/featured/`). Then add one or two per product line, favoring anything described as a bestseller.
4. **Plan the rows.** The rows on the storefront homepage come from collections. Without them, a buyer sees the banner and a few category tiles, and nothing else. Plan 3 to 5:
   - "Featured Products" (`featured: true`) always goes first.
   - Then add one row per product line, using `product_subcategory_ids` or `product_category_ids`. You can also group by `brand_ids` or effect (`product_group_ids`).
   - Multi-brand on one storefront: one row per brand (`brand_ids`), right after "Featured Products".
   - A product joins a row when it matches every filter you set. Within one filter, matching any value is enough. Empty rows are hidden.
5. **Build it.** One call to the onboarding API:

```bash
curl -s -X POST $ONBOARD/storefront -H 'Content-Type: application/json' -d '{
  "account_token": "'$ACCOUNT_TOKEN'", "name": "Acme Wholesale",
  "logo_url": "https://.../acme-logo.png", "banner_url": "https://.../acme-banner.jpg",
  "accent_color": "#B4532A", "description": "Small-batch fruit gummies ...", "website": "https://acme.example",
  "collections": [
    {"name": "Featured Products", "featured": true},
    {"name": "THC Gummies", "product_category_ids": ["51104"]},
    {"name": "Hemp CBD Gummies", "product_subcategory_ids": ["64724"]},
    {"name": "CBD Sparkling Waters", "product_subcategory_ids": ["64725"]}]}'
# → {"menu_id": "812", "menu_ref": "...", "url": "https://staging.distru.com/menu/acme-edibles/acme-wholesale", "rows": [...]}
```

Storefront notes:

- **Optional fields.** `location_ids` (the default is your first location), `url_suffix` (the default comes from `name`), and `hide_prices`.
- **Prices.** `hide_prices` is `HIDE_FOR_NON_LOGGED_IN_USERS` by default. Use `HIDE_FOR_EVERYONE` while prices are $0, or `DO_NOT_HIDE` once real prices are in.
- **Ordering before stock.** Retailers can order before stock is loaded. The call turns that on for you.
- **Images.** `logo_url` and `banner_url` must be public image URLs. The LeafLink brand payload gives you both.
- **Changing it later.** Send the same call with `menu_id` and `menu_ref` from the first response. `collections` replaces the whole list, so always resend every row.

### Multi-brand

**One storefront for all brands (recommended).** Make one `/api/storefront` call. Put "Featured Products" first, then one row per brand with `brand_ids`, then product lines if there's room. Keep it to 8 rows or fewer. Use the company's logo, or the lead brand's if the company has none.

**One storefront per brand.** Every product starts on every storefront (`INCLUDE_IN_ALL`), so build the storefronts first, then pin each product to its brand's storefront.

1. Call `/api/storefront` once per brand, with that brand's `name`, `logo_url`, `banner_url`, `accent_color`, `description`, and `website`. Send `url_suffix` in plain lowercase ASCII (`fernwood-wholesale`), because accented letters turn into dashes. Keep each response's `menu_id` and `menu_ref`.
2. Inside each storefront, put "Featured Products" first, then that brand's product lines. Feature 2 to 4 products per brand.
3. Pin each product to its brand's storefront: `POST /products {"id":"<product id>","menu_visibility":"INCLUDE_IN_SELECT","menus":["<menu_id>"]}`. *Unverified.* If the public API rejects the id, read the storefront ids from `GET /menus`.
4. Open each link and check that it shows only its own brand.

All of an account's storefronts share one link prefix (`/menu/<account-slug>/`), so `url_suffix` is what tells them apart. On LeafLink, each brand's own brand payload has its logo and banner (leaflink.md §2).

## §7 Gotchas we hit

- Numeric ids are rejected. Send strings.
- A category without `official_product_category_id` fails with "Select a category type".
- Product names reject `+` and commas.
- A storefront with no collections looks empty: only category tiles, no product rows.
- The storefront needs a location first. Without one, `/api/storefront` returns 409 and says so.
- Every `/api/accounts` call makes a new account. Don't retry it blindly.
