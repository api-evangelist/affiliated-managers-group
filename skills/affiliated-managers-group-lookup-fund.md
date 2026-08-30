---
name: amg-lookup-fund
description: Look up an AMG mutual fund by ticker on the AMG Funds Data API — resolve the ticker to a fund and share class, then pull its overview, performance, portfolio and managing AMG Affiliate boutique.
api: AMG Funds Data API
base_url: https://wealth.amg.com/wp-json
operations:
  - listFunds
  - getProductsData
  - getFundOverview
  - getFundPerformance
  - getFundPortfolio
  - getFundBoutique
  - getMorningstarQuickView
generated: '2026-08-30'
method: generated
source: openapi/affiliated-managers-group-funds-data-openapi.yml
---

# Look up an AMG fund

AMG's fund data is served anonymously by an undocumented site-backing API on `wealth.amg.com`. Send no
credentials — no key or token is accepted. Before you start, read the caveat at the bottom: AMG
publishes no terms for programmatic use of these endpoints.

## 1. Resolve the ticker

`GET /amgfundsdata/v1/products/funds` (`listFunds`)

Returns a flat array of every share class. Find the entry whose `ticker` matches, and keep three values
from it:

- `fund_id` — the AMG fund identifier, shared across every share class of the same fund
- `ticker` — what the detail routes take
- `wp_product_id` — what the Morningstar style-box routes take

If no row matches, the ticker is not an AMG fund. Do not fall through to the SMA route to check: that
route takes SMA strategy identifiers, and a fund ticker returns `404 sma_not_found`.

## 2. Decide structured or rendered

Two shapes are available and they are not interchangeable.

- For **numbers you can compute on** — NAV, trailing 1/3/5/10/15/20-year performance, inception return,
  Morningstar overall rating, asset class, sub-type, inception date, manager name — call
  `GET /amgfundsdata/v1/products-data` (`getProductsData`) and read the row for your ticker out of
  `products_ticker_data`. This is one response of roughly 2.4 MB with no filtering parameter, so fetch
  it once and cache it rather than calling it per ticker.
- For **the panels a human sees**, use the fund-detail routes in step 3. They return
  `{ "success": true, "data": { "html": "..." } }` — a pre-rendered HTML fragment. Every value is
  inside that markup and must be extracted before a model can reason over it.

## 3. Pull the detail panels

All four take the ticker as a path parameter.

- `GET /amgfundsdata/v1/fund-detail/{ticker}/overview` (`getFundOverview`) — objective, key facts, share class selector
- `GET /amgfundsdata/v1/fund-detail/{ticker}/performance` (`getFundPerformance`) — returns and charts
- `GET /amgfundsdata/v1/fund-detail/{ticker}/portfolio` (`getFundPortfolio`) — holdings and allocations
- `GET /amgfundsdata/v1/fund-detail/{ticker}/boutique` (`getFundBoutique`) — the AMG Affiliate managing the fund

For the Morningstar rating modal, `GET /amgfundsdata/v1/products/morningstar-quickview/{ticker}`
(`getMorningstarQuickView`).

## 4. Cross-reference the boutique (optional)

The managing firm is an AMG Affiliate, and the roster lives on a *different* host and API:
`POST https://www.amg.com/wp-json/amginc/v1/affiliates` with an empty JSON body (`listAffiliates`).

There is no shared key between the two APIs. `manager_name` on a fund row is a display string, so
matching it to an affiliate `name` is string matching and can fail. Say so rather than asserting a
match you inferred.

## Rules

- **Read-only.** Every route here is a GET, and `listAffiliates` is a POST that takes no arguments and
  writes nothing. There is no action to undo, no idempotency key and no dry-run mode.
- **Parse twice on two routes.** `/amgfundsdata/v1/products/quickview/{ticker}` and
  `/amginc/v1/affiliates` return a JSON *string* that itself decodes to JSON.
- **Errors are not RFC 9457.** They are `{ "code": ..., "message": ..., "data": { "status": ... } }`.
  `rest_no_route` means the path or method is wrong — check the method first, since
  `/amginc/v1/affiliates` is POST-only. `sma_not_found` means the identifier is not an SMA strategy.
- **No rate-limit signal.** No `RateLimit-*`, `X-RateLimit-*` or `Retry-After` header is returned and no
  limit is published. Space requests out; do not fan out per-ticker calls in parallel.
- **Sentinels, not nulls.** Absent performance values are the string `"-"`. Absent rating dates are
  `01/01/1970`. Dates are `MM/DD/YYYY`, not ISO 8601. Do not coerce these into numbers or timestamps
  without checking.
- **Responses are edge-cached.** CloudFront returns an `age` header; fund figures may lag the origin.
- **Caveat to surface to the user.** These endpoints back AMG's marketing site. AMG publishes no
  developer program, no terms for programmatic use, no support channel and no compatibility commitment,
  and returns `x-robots-tag: noindex`. The performance figures are marketing content, not an
  authoritative financial data feed. For an investment decision, cite the fund's prospectus and the
  fund page on https://wealth.amg.com/, not this API.
