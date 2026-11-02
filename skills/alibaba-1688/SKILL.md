---
name: alibaba-1688
description: 1688 (1688.com, 阿里巴巴1688) and Alibaba.com wholesale sourcing data through the alibaba-1688 MCP tools. Use when the user wants 1688 or Alibaba.com products, factories or suppliers found by keyword, wholesale prices, price tiers, MOQ (起批量) or dropshipping (一件代发) compared, 1688 product variants read by offer ID, Alibaba.com supplier profiles checked, or Alibaba.com supplier store reviews read. Not for Alibaba Cloud.
---

# 1688 & Alibaba.com (阿里巴巴1688 / 阿里巴巴国际站)

Three sourcing tools plus Apify's run and dataset tools, served by the `alibaba-1688` MCP server. Each sourcing tool starts a paid run on the user's Apify account and returns run metadata. The rows arrive in a dataset. Every request follows the **run flow** below.

| Tool (client prefixes vary) | Use for |
|---|---|
| `zen-studio--1688-wholesale-scraper` | 1688 products by keyword or offer ID: price, quantity tiers, MOQ, sales, repeat-buyer rate, supplier, shipping; optional variant matrix |
| `zen-studio--alibaba-scraper` | Alibaba.com products (keyword or category) with price ladder, MOQ and supplier profile, or supplier profiles directly |
| `zen-studio--alibaba-supplier-review-scraper` | Every store review of Alibaba.com suppliers, with the store's rating breakdown |
| `get-actor-run` | Wait for and check a run |
| `get-dataset-items` | Read the rows |
| `fetch-actor-details` | Full input reference and README of any of the three |
| `get-key-value-store-record` | Read a record a run saved, such as the Alibaba.com `OUTPUT` record with filter values |
| `abort-actor-run` | Stop a run the user no longer needs |

If only Apify's generic `call-actor` is available, call it with `actor: "zen-studio/<name>"` (for example `zen-studio/1688-wholesale-scraper`) and the same input; the run flow is identical.

"Alibaba" in a sourcing question means Alibaba.com, the B2B marketplace. These tools do not cover Alibaba Cloud (Aliyun).

## Run flow

A tool call returns after at most 45 seconds, whether or not the run has finished. Many runs take longer, so treat the first response as a receipt, not the result.

1. **Start.** Call the tool with the recipe input and `"waitSecs": 45`. Keep the `runId` from the response. Done when you hold a `runId`.
2. **Poll.** Read `status`. While it is `READY` or `RUNNING`, call `get-actor-run` with `runId` and `waitSecs: 45`, and tell the user the run is still going between calls. Done when `status` is terminal: `SUCCEEDED`, `FAILED`, `ABORTED` or `TIMED-OUT`. A response that is already `SUCCEEDED` skips straight to step 3; small searches usually finish within the first call.
3. **Fetch.** Call `get-dataset-items` with `datasetId` from `storages.datasets.default.id`. Pass `limit` (20 to 50) and a `fields` projection from **Reading results**; 1688 rows carry 60+ fields and Alibaba.com rows several hundred. A dotted field such as `supplier.companyName` comes back as a flat key named `"supplier.companyName"`; a top-level field such as `quantityPrices` comes back whole. Page with `offset` until you have what the question needs. Done when every row the answer depends on is read.
4. **Answer.** Summarise for the user: prices with their currency, MOQ with its unit, supplier names and links (`detailUrl`, `product.url`, `supplier.homeUrl`), and anything filtered out. Translate Chinese titles when the user writes in another language. When comparing 1688 with Alibaba.com, convert to one currency and say which rate you used.

One run per request. A run that is still `RUNNING` keeps working and billing; starting a second run for the same question pays twice. Resume polling the same `runId` instead, including after a client timeout or a new turn.

Terminal states other than `SUCCEEDED`: `TIMED-OUT` and `ABORTED` can hold partial rows, so fetch them and say the result is partial. `FAILED`: report `statusMessage` and change the input before retrying. A `SUCCEEDED` run whose item count reads 0 can lag for a few seconds; fetch once before concluding it is empty.

## Chaining the tools

- **1688 search, then the full product.** Pass a row's `offerId` to `offerIds` with `includeSkuDetails: true` for every colour and size with its own price and stock.
- **Alibaba.com product, then reviews.** Pass `supplier.homeUrl` (or `product.url`) to the review tool's `supplierUrls`. A company name also works through `companyNames`, but a link is exact.
- **Repeat Alibaba.com supplier searches.** Pass the `supplier.companyId` values the user already has as `excludeIds`; they are skipped and not billed.
- **1688 and Alibaba.com together.** Search 1688 in Chinese and Alibaba.com in English for the same product (for example `不锈钢水杯` and `stainless steel water bottle`). The two marketplaces have separate suppliers; do not claim a 1688 factory is the same company as an Alibaba.com supplier unless the names match.

## Input recipes

Start small (10 to 30 results) unless the user asks for more; every returned product, supplier and review is billed.

**1688 products** (`1688-wholesale-scraper`):

```json
{ "keywords": ["不锈钢水杯"], "maxResults": 20, "sortBy": "bestSelling",
  "priceMax": 15, "minOrderQuantity": 100 }
```

- `keywords`: a list; each keyword has its own `maxResults` budget. The default `maxResults` is 100; always set it. Chinese keywords return more and better matches than English; translate the user's product name.
- `offerIds`: numeric 1688 product IDs (from `offerId=` or `/offer/<id>.html` in a link) instead of keywords. Each ID gets a row; one that cannot be returned has an `error` field saying whether to retry. Search filters and `maxResults` do not apply to ID lookups, and ID rows lack `orderCount` and `repurchaseRate`: keep those values from the search that found the product. On the free plan an ID lookup uses one of the 25 runs, like a search.
- `sortBy`: `relevance` (default), `bestSelling`, `priceAsc`, `priceDesc`.
- `priceMin`, `priceMax`: unit price in yuan (whole numbers). `minOrderQuantity`: only offers whose MOQ is at most this.
- `merchantType`: `any`, `superFactory` (超级工厂), `verifiedMerchant` (实力商家). `province` and `city` take Chinese names: `浙江`, `广东`, `义乌市`, `深圳市`.
- `excludeKeywords`: drops products whose title contains any of these words, before billing. Use it when a keyword has a second meaning.
- `includeSkuDetails: true` adds the variant matrix (`skuDetails.variants` with price, stock, weight per variant). Use it for offer-ID lookups or small searches.
- `includeSupplierIntelligence: true` adds dropship metrics, resale channels, certificates, compliance numbers and package sizes. It is billed per product on top of the base price; enable it only when the user asks for that data.

**Alibaba.com products and suppliers** (`alibaba-scraper`):

```json
{ "resultType": "products", "keywords": ["stainless steel water bottle"],
  "maxResults": 20, "shipToCountry": "US", "verifiedManufacturer": true }
```

- `resultType`: `products` (default, listings with the supplier attached), `suppliers` (company profiles), `category` (browse a whole category; needs `category`, no keyword).
- `shipToCountry` (default `US`) localizes prices, local stock and delivery; set it to the user's market. `marketplace: "worldwide"` applies to products only.
- Filters for keyword searches: `sortBy` (`relevance`, `sales_volume`), `minPrice`, `maxPrice` (ship-to currency), `maxMOQ`, `supplierCountries` (ISO codes), `minReviewScore` (`4`, `4.5`, `5`), `verifiedSupplier`, `verifiedManufacturer`, `tradeAssurance`, `verifiedProSupplier`, `companyAuthTags` (ISO, BSCI), `productCertTags` (CE, ROHS), `maxDeliveryDays`, `localStock`, `paidSamples`. Suppliers mode honours only verification, MOQ and country.
- `includeReviews: true` with `maxReviewsPerProduct` adds a capped sample of store reviews per row; `includeSupplierReport: true` adds verification reports for verified manufacturers. Both are billed per item. For a supplier's full review history use the review tool instead.
- A product several keywords find is returned and billed once per run.
- `maxResults` defaults to 100 **per keyword**, so three keywords can bill up to 300 products. Always set it. For `resultType: "category"` it is the total only when no keywords are given; with keywords each keyword gets its own budget.

**Supplier reviews** (`alibaba-supplier-review-scraper`):

```json
{ "supplierUrls": ["https://ansune1.en.alibaba.com/"], "maxReviewsPerSupplier": 30 }
```

- `supplierUrls`: storefront links (`https://<name>.en.alibaba.com`) or any Alibaba.com product link (its supplier is used). `companyNames`: the full legal name works best; a name no supplier carries returns nothing.
- Always set `maxReviewsPerSupplier`: the default is 100 and `0` means every review.
- One row per review; `storeAggregate` (average, totals, per-dimension distribution) repeats on each row of a supplier.

## Reading results

Project fields with `get-dataset-items` `fields`. Good starting sets:

- 1688 products: `error,message,limit_reached,retry_later,offerId,title,price,quantityPrices,minOrderQuantity,unit,saledCount,soldDisplay,orderCount,repurchaseRate,province,city,supplier.companyName,supplier.sellerType,supplier.bizType,dropship.enabled,detailUrl`
- 1688 variants: `error,message,limit_reached,retry_later,offerId,title,price,skuDetails`

Read `limit_reached`, `retry_later` and `error` first. Rows with `limit_reached` or `retry_later` are notices, not products. A row with `error` and no `price` is a failed lookup; a row with `error` and a `price` is a usable partial product: keep its fields and mention the warning.
- Alibaba.com products: `product.productId,product.title,product.price,product.moq,product.soldOrder,product.url,detail.ladderPrices,detail.moq,detail.sample,supplier.name,supplier.country,supplier.goldSupplierYears,supplier.homeUrl,supplier.companyId,companyProfile.averageStar,companyProfile.supplierResponseTime,companyProfile.SupplierOnTimeDeliveryRate`
- Alibaba.com suppliers: `supplier.name,supplier.homeUrl,supplier.companyId,supplier.profile,supplier.tagList,supplier.productList,companyProfile.averageStar,companyProfile.companyLocation`
- Supplier reviews: `reviewContent,rating,ratingType,reviewTime,buyer.countryName,product.productName,supplier.companyName,storeAggregate.averageScore,storeAggregate.totalReviews`

Field notes:

- 1688 prices are CNY. `price.basis` is `quantity` (tiers in `quantityPrices`) or `variant` (one price per variant; read `skuDetails.variants` for exact prices). A listed price of ¥1 can be a placeholder for a quote; check the variant label.
- `repurchaseRate` and `returnRate` both hold the shop's repeat-buyer rate (回头率, 0.61 = 61%); `returnRate` is not a refund rate. `orderCount` is 30-day orders, `saledCount` lifetime units sold.
- `supplier.sellerType` reads `super_factory`, `normal_factory` and similar; `supplier.bizType` reads `生产加工` (manufacturer) or a trading type.
- Alibaba.com `product.price` and `product.moq` are display strings (`"$2.99-4.94"`, `"Min. order: 100 pieces"`); `detail.moq` is the number and `detail.currency` its currency.
- Neither marketplace returns phone numbers or email addresses. Point the user to `supplier.shopUrl` (1688) or `supplier.homeUrl` (Alibaba.com).

`fetch-actor-details` returns the full field reference when the user asks for something outside these sets. Alibaba.com keyword runs can also save an `OUTPUT` record (`availableAttributeFilters`) in their key-value store; read it with `get-key-value-store-record` (`keyValueStoreId` from the run's `storages.keyValueStores.default.id`, `recordKey: "OUTPUT"`) when the user wants the filter values a search offers.

## Cost and limits

- Pricing is per result on each tool's Apify Store page (`https://apify.com/zen-studio/<name>/pricing`) and is billed to the user's Apify account. Quote the page rather than a remembered number.
- On Apify's free plan:
  - 1688 products: 25 runs in total, up to 20 products per run, first keyword only, up to 20 offer IDs. The run's `statusMessage` says so ("Free plan preview: up to 20 results per run (N free run(s) remaining)."). When the runs are used up, the dataset holds one row with `limit_reached: true`. A row with `retry_later: true` ("The free preview is temporarily unavailable...") is a short hold: try again later.
  - Alibaba.com products, suppliers and supplier reviews: no separate free-plan cap. Runs are billed per result against the user's Apify credit, so always set `maxResults`, `maxReviewsPerProduct` and `maxReviewsPerSupplier`.
  - Supplier reviews: no separate free-plan cap; every review is billed against the user's Apify credit, so always set `maxReviewsPerSupplier`.
  - Tell the user when a result is capped or empty for this reason, and that any paid Apify plan removes the caps. Free runs are scarce, so do not spend them on tests.
- `maxResults`, `maxReviewsPerProduct` and `maxReviewsPerSupplier` are the cost controls. State the caps you chose in the answer.
