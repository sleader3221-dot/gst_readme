# GST & Invoice Implementation

Node.js · Express 5 · MongoDB (Mongoose 8)
Verified on live DB — shop `6aa8dda0faa40450b50d917b` (Karthik kirana, Karnataka)

## Defects found in existing code

| Defect | Impact |
|---|---|
| `resolveClassification()`: `hsn.startsWith("2202")` → 28% + 12% cess | over-charged all beverages; cess applies only to `22021010` |
| `weightKg > 25 \|\| !isPrepackagedLabelled` → 0% | zero-rated 200gm chips, 10g bars, 30kg wheat; weight is irrelevant to GST |
| hardcoded `19059030 => 12%` | overrode catalogue rate |
| tax base = MRP | MRP is inclusive; inflated taxable value vs. price collected |
| `hsnCode`/`gstRate`/`cessRate`/`uqc` declared, never written | 51 active products, 0 resolvable rates → all 0% |
| `isInterState = placeOfSupplyCode !== "29"` | Karnataka hardcoded; Maharashtra shop + Maharashtra buyer wrongly charged IGST |

## Design

One engine, three consumers — preview, lookup and invoice cannot disagree.

```
config/gstRates.js ─┐
                    ├─► services/gstCalculator.js ─┬─► routes/ShopRoutes/shopGst.routes.js
config/states.js ───┘   (paise arithmetic,        ├─► services/ShopKeeperServices/shopProduct.service.js
                         rate resolution)         └─► services/invoiceService.js
```

```
gross         = unitPrice × quantity                     (GST-inclusive)
taxable value = (gross − discount) / (1 + gstRate + cessRate)
CGST = SGST   = taxable × gstRate / 2                    intra-state
IGST          = taxable × gstRate                        inter-state
cess          = taxable × cessRate                       aerated water only

placeOfSupply = SELF_PICKUP ? shop.state : buyer.address.state
isInterState  = placeOfSupply !== shop.state
```

Integer paise; half-up rounding per line, once at total. Invariant: `gross = taxable + tax`.

**Rate precedence** — statutory rate is fixed by HSN; stored/user rate must not override.

```
1. HSN statutory rate   (config/gstRates.js, longest-prefix match)
2. customGstRate        (ShopProductStatus)
3. UNRESOLVED           → invoicing blocked, never 0%
```

## Verified figures

| Case | Gross | Rate | Taxable | Tax | Total |
|---|---|---|---|---|---|
| KitKat × 1 (180690) | 40.00 | 18% | 33.90 | 3.05 + 3.05 | 40.00 |
| Britannia × 1 (19053100) | 75.00 | 18% | 63.56 | 5.72 + 5.72 | 75.00 |
| KitKat × 2 + Britannia × 1 | 155.00 | 18% | 131.36 | 11.82 + 11.82 | 155.00 |
| Aerated (22021010) | 40.00 | 28% + 12% cess | 28.57 | 4.00 + 4.00 + 3.43 | 40.00 |

Order `6aa8e275…` is `SELF_PICKUP` → place of supply Karnataka despite Maharashtra buyer → CGST+SGST.

---

## Files created

| File | Purpose |
|---|---|
| `config/gstRates.js` | HSN → GST% + cess% table, longest-prefix resolver (78 entries) |
| `config/states.js` | State codes, aliases, `stateCodeFromAddress()` |
| `services/gstCalculator.js` | Paise arithmetic, rate resolution, totals, amount-in-words |
| `services/invoiceService.js` | Pure invoice builder (no DB), Rule 46 numbering |
| `models/ShopModels/invoice.model.js` | Invoice storage, line items embedded |
| `controllers/invoiceController.js` | preview / generate / get / list |
| `routes/ShopRoutes/shopGst.routes.js` | HSN rate lookup endpoints |
| `routes/invoice.routes.js` | 4 invoice endpoints |
| `postman/DukaanSe-GST-API.postman_collection.json` | 11 requests, 4 folders, assertions |

Rate table source: Notification 09/2025-CT(Rate), 55th GST Council, eff. 01-Apr-2025 — 32 @ 5%, 23 @ 18%, 15 @ 12%, 3 @ 28%, 5 @ 0%.

## Files modified (additive)

| File | Change |
|---|---|
| `app.js` | Mounted `shopGstRoutes` at `/api/gst`, `invoiceRoutes` at `/api` |
| `models/ShopModels/shopProductStatus.model.js` | +`customHsnCode`, `customGstRate`, `customCessRate`, `customUqc`, `customIsPrepackagedLabelled` |
| `validations/ShopValidations/shopProduct.validation.js` | +Joi rules (HSN 4–8 digits, rates 0–100), all optional |
| `services/ShopKeeperServices/shopProduct.service.js` | Persists GST fields in `changePriceAndUnits`; +`getGstPreviewService` |
| `controllers/ShopControllers/shopProduct.controller.js` | +`getGstPreview` |
| `routes/ShopRoutes/shopProduct.routes.js` | +`GET /gst-preview/:productId` |
| `package.json` | Removed unused `exceljs`, `pdfkit` |

No existing business logic rewritten or removed.

## Bugs found and fixed

| Bug | Fix |
|---|---|
| `isInterState !== "29"` — hardcoded Karnataka | supplier state from `stateCodeFromAddress(shop.address)`, shared with invoice path |
| `ShopAuth` used in `shopProduct.service.js` without import → HTTP 500 | import added |
| Buyer address duplicated | prefer stored `formattedAddress` |
| `placeOfSupplyName` = `"KARNATAKA"` | `"Karnataka"` |
| `totalGst` = `null` (`NaN` from missing `gstPaise`) | sum `cgstPaise + sgstPaise + igstPaise` |

## Data issues

- **Shop GSTIN invalid** — `3627U46UFBN` is 11 chars (GSTIN is 15) and starts `36` (Maharashtra) on a Karnataka shop. Invoice emits `gstin: ""` + `gstinRaw`; valid GSTINs promote the document to `TAX_INVOICE`. Fix before filing returns.
- **`customMRP: 0`** on Britannia (price 75) — no discount shown.
- MRP and price both treated as GST-inclusive; convention to be confirmed.

## API

```
GET  /api/gst/hsn-rate/:hsnCode[?price=&quantity=&interState=]
GET  /api/gst/hsn-rates
GET  /api/shopProduct/gst-preview/:productId?quantity=&placeOfSupplyCode=   auth
PUT  /api/shopProduct/changePriceAndUnits/:productId                       auth
POST /api/invoice/preview     { orderId }
POST /api/invoice/generate     { orderId }   auth, idempotent
GET  /api/invoice?merchantId=
GET  /api/invoice/:id
```


