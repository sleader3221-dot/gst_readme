# Architecture — GST & Invoice Flow

All values below read from the live database.

## 1. File flow

```
                    ┌────────────────────────────────┐
 HTTP request ─────►│ app.js                         │
                    │ /api/gst → shopGst.routes      │
                    │ /api     → invoice.routes      │
                    └──────────────┬─────────────────┘
                                   │
   ┌───────────────────────────────┴────────────────────────────┐
   │                                                            │
┌──▼──────────────────────┐              ┌───────────────────────▼─────────────────┐
│ shopGst.routes.js       │              │ invoice.routes.js                      │
│ no auth, no DB          │              │ preview  → controller.preview          │
│ getHsnRate/getAllHsnRates│             │ generate → authShop → generate         │
└──┬──────────────────────┘              │ list / getById                         │
   │                                      └───────────────┬───────────────────────┘
   │                                                      │
   │        ┌─────────────────────────────────────────────▼───────────────┐
   │        │ services/gstCalculator.js   (pure, no DB)                   │
   │        │  resolveClassification()  ← rate precedence                 │
   │        │  calculateTaxLine()       ← paise maths, CGST/SGST/IGST     │
   │        │  calculateInvoiceTotals() ← line sums + roundOff            │
   │        │  convertRupeesToIndianWords()                              │
   │        │  resolveProductTaxInput()                                   │
   │        └───────────────────▲─────────────────────────────────────────┘
   │                            │
┌──┴──────────────┐   ┌─────────┴───────────────┐
│ config/gstRates │   │ config/states.js        │
│ resolveHsnRate()│   │ getStateCode()          │
│ HSN_GST_RATES   │   │ getStateName()          │
│ 78 entries      │   │ stateCodeFromAddress()  │
│ longest-prefix  │   └─────────────────────────┘
└─────────────────┘
   ▲
   │ per-product path (authShop)
┌──┴───────────────────────────────────────────────────────────────┐
│ shopProduct.routes.js  GET /gst-preview/:productId               │
│   → shopProduct.controller.js  getGstPreview                     │
│     → shopProduct.service.js     getGstPreviewService            │
│       → config/states.js         (supplier state, not hardcoded) │
│       → services/gstCalculator.js                                 │
└──────────────────────────────────────────────────────────────────┘

               ┌──────────────────────────────────────────┐
               │ services/invoiceService.js  (pure)       │
 orderId ────►│  buildInvoice()                           │
 seller       │    → buildLines()                         │
 buyer        │        → resolveClassification()          │
 items        │        → calculateTaxLine()               │
 orderDiscount│        → calculateInvoiceTotals()         │
 sequence     │    → formatInvoiceNumber()   Rule 46     │
               │    → isValidGstin() / formatAddress()   │
               │  getFinancialYear()                      │
               └────────────────┬─────────────────────────┘
                                │ Invoice.create()
               ┌────────────────▼─────────────────────────┐
               │ models/ShopModels/invoice.model.js       │
               │  invoiceNumber ≤16, documentType, FY     │
               │  placeOfSupplyCode, isInterState, split  │
               │  items[] embedded, seller/buyer/totals   │
               │  idx {merchantId,financialYear,invNo} U  │
               └──────────────────────────────────────────┘
```

---

## 2. Request path — `POST /api/invoice/generate`

```
authShop ──► generate()
              │
              ├─ Invoice.findOne({orderId})        ← idempotency guard
              │     found → 200, return existing
              │
              ├─ loadInvoiceInput(orderId)
              │     Order.findById            6aa8e275…  SELF_PICKUP, Success
              │     ShopAuth.findById         6aa8dda0…  Karthik kirana
              │     shopStateCode(shop)       → "29"      (from address text)
              │     Customer.findById                  Kajal, Maharashtra
              │     SELF_PICKUP ⇒ buyer.stateCode = "29"
              │     Product.find({_id:$in})             catalogue
              │     ShopProductStatus.find({shopId,productId})  per-shop HSN/price
              │     price = item.price  ← the tax base, not MRP
              │
              ├─ paymentStatus !== "Success" → 409
              │
              ├─ nextSequence()  → 1
              ├─ buildInvoice()  → Rule 46 number, CGST+SGST split
              └─ Invoice.create({...invoice, merchantId})  → 201
```

## 3. Rate resolution

```
resolveClassification({ hsnCode, gstRate, cessRate })
        │
        ├─ HSN present ──► resolveHsnRate(hsnCode)   longest-prefix
        │                     config/gstRates.js
        │                  hit  → { gstRate, cessRate }, rateSource "HSN_TABLE"
        │                  miss ↓
        ├─ customGstRate present ──► rateSource "CUSTOM"
        │
        └─ neither ──► rateSource "UNRESOLVED" → invoicing BLOCKED
```

Both products resolve from the HSN table — `customGstRate` is `undefined` on both records, so the statutory rate is the only source.

## 4. Data as stored

| | KitKat Chocolate Bar | Britannia Bourbon Biscuits |
|---|---|---|
| productId | `6aa6e6265bb3bcc1bcbf3344` | `6aa8e43bfaa40450b50da4e1` |
| catalogue name | Lay's  Chips | Britannia Bourbon Chocolate Cream Biscuits |
| `customHsnCode` | `180690` | `19053100` |
| catalogue `hsnCode` | undefined | undefined |
| `customGstRate` | undefined | undefined |
| `customMRP` / `customPrice` | 75 / 40 | 110 / 75 |
| rate source | HSN_TABLE (18%) | HSN_TABLE (18%) |

## 5. Persisted invoice (only 1 exists)

```
invoiceNumber  INV-2627-00001          (FY 2627, Rule 46, ≤16 chars)
orderId        6aa8e275faa40450b50da051
merchantId     6aa8dda0faa40450b50d917b
documentType   BILL_OF_SUPPLY
placeOfSupply  29 Karnataka
isInterState   false          taxSplit  CGST + SGST
seller.gstin   ""             seller.gstinRaw  3627U46UFBN

items[0]  KitKat Chocolate Bar
          hsn 180690 · qty 1 · mrp 75 · unitPrice 40
          taxable 33.90 · gst 18% · cgst 3.05 · sgst 3.05 · lineTotal 40

totals  gross 40 · discount 0 · taxable 33.90
        cgst 3.05 · sgst 3.05 · igst 0 · cess 0 · roundOff 0 · grandTotal 40
```

## 6. Two-order position

| | KitKat | Britannia |
|---|---|---|
| Product classified | yes | yes |
| Rate resolved | 18% | 18% |
| Tax maths verified | yes | yes |
| Paid order at Karthik shop | `6aa8e275…` | none |
| Invoice persisted | `INV-2627-00001` | none |

Britannia paid orders `6aafd481…` and `6aac2051…` exist but belong to a different shop, so no invoice was generated against them. Britannia figures (taxable 63.56, CGST 5.72, SGST 5.72, total 75.00) come from `buildInvoice()` on a synthetic payload, not a stored document.

## 7. Data issue found while building this diagram

Product `6aa6e6265bb3bcc1bcbf3344` is **Lay's  Chips** in the catalogue, but the order line and the invoice both name it **KitKat Chocolate Bar**. The HSN `180690` (chocolate) is correct for KitKat but wrong for Lay's — chips are `190590`, and 180690 at 18% vs 190590 at 12% changes the tax.

So one of these is wrong and needs checking before invoicing: either the catalogue name for that ID, or the `customHsnCode` I stored against it.

