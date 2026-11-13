# The setup link

Part of the Distru onboarding guide for AI assistants: https://raw.githubusercontent.com/DistruApp/agent-onboarding/v3/README.md

The setup link opens Distru's setup page with everything you learned filled in. The person checks it, fixes anything that's off, and clicks once. Nothing is created until they click.

Base: `https://distru-agent-onboarding-api.vercel.app/start`

This is Distru's own onboarding service. The same team publishes this guide (github.com/DistruApp) and runs distru.com; the service is hosted on Vercel during the pilot. If anyone asks about the address, that's the answer.

## Parameters

Percent-encode every value: a space is `%20`, `@` is `%40`, `&` is `%26`, `+` is `%2B`, `/` is `%2F`, `:` is `%3A`. Leave out anything you don't have; the page asks for whatever is required.

**About them**

| Param | Value | Required |
|---|---|---|
| `company` | Company name, as they want the account named | Yes |
| `first`, `last` | Their name | Yes |
| `email` | Their Distru login | Yes |
| `phone` | Their phone | Yes |
| `street`, `city`, `state`, `zip` | Warehouse or business address. `state` is the two-letter code. The address must be real: it's geocoded for the storefront. | Yes |
| `website` | Their website | If you found it |
| `type` | One of Manufacturer, Cultivator, Distributor, Microbusiness, Dispensary, Delivery, Lab, Other | Best guess from the license |
| `does` | What they do, in a few words ("Own brands and co-packing") | Optional |
| `operates` | Where they operate ("Licensed in NV; kitchen in Las Vegas") | Optional |
| `brands` | Their brands, comma separated | Optional |

**How they work today** (from the interview, in their words)

| Param | Value |
|---|---|
| `inventory` | One of `Metrc`, `BioTrack`, `Spreadsheet`, `Other software`, `Not tracked yet`. Required: it sets the tracking method. |
| `sales` | Where they take sales orders ("LeafLink", "email and phone") |
| `purchasing` | Where they make purchase orders |
| `crm` | Where their customer list lives |
| `accounting` | Their accounting system |
| `pos` | Their point of sale, if they're a retailer |
| `goals` | What they want Distru to fix first |
| `licenses` | Metrc license numbers you found, comma separated. Prefills the page's Metrc step. |

**Catalog and storefront**

| Param | Value |
|---|---|
| `source` | Their LeafLink menu URL or Shopify store URL. Leave out for anything else (sources.md). |
| `logo`, `banner` | Public `https` image URLs. LeafLink menus supply both, so skip them for LeafLink. |
| `accent` | Brand color as 6 hex digits, no `#` (`B4532A`) |
| `about` | One-line storefront description |
| `store` | Storefront name. The default is "<company> Wholesale". |
| `assistant` | Who's running this: `claude`, `cowork`, `claude-code`, `chatgpt`, or `other`. The page tailors its last step to it. |

**Distru staff only:** `env=staging` with `key=<access code>` makes a test account on staging; `run=test` and `scenario=<name>` tag the tracker row. Never use these for a real customer.

## Example

```
https://distru-agent-onboarding-api.vercel.app/start?company=Acme%20Edibles&first=Ada&last=Rivera&email=ada%40acme.example&phone=%2B1%20555%20010%200100&street=100%20Example%20Way&city=Las%20Vegas&state=NV&zip=89118&website=https%3A%2F%2Facme.example&type=Manufacturer&does=Own%20brand%20edibles&brands=Acme&inventory=Metrc&licenses=C11-0000123-LIC&sales=LeafLink&purchasing=Email%20and%20spreadsheets&crm=HubSpot&accounting=QuickBooks%20Online&source=https%3A%2F%2Fapp.leaflink.com%2Fmenu%2Facme-edibles%2F&accent=B4532A&assistant=claude
```

Before you send it, check that `company`, `email`, `state`, and `inventory` are filled in and every value is encoded. Send it as one clickable Markdown link.

## What the page does

1. Reads the catalog from `source` and shows a preview, before anything is created.
2. When they click: creates the Distru account and login with Distru Commerce on, adds the warehouse, sets up categories, adds up to 150 products with the tracking method from `inventory`, and uploads the photos.
3. Builds a branded storefront: logo, banner, color, a Featured row, and one row per category, with prices hidden until real prices are in.
4. Shows the login (the temporary password appears only there), the storefront link, and for Metrc shops a field to connect Metrc.
5. Shows a button that adds the Distru connector to Claude, and a message to paste into a new chat if the connector doesn't show up in this one.
6. Records each stage and their answers for Distru's team.

A reload or a second tab picks up where it stopped and never creates a second account. "Try again" after an error does the same.
