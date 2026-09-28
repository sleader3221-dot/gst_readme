# GST & Invoice — Flow

One direction, top to bottom. Every path ends in the same place: the tax engine.

## A. Invoice flow (main path)

```
STEP 1   REQUEST
         POST /api/invoice/generate   { orderId }
         │
         ▼
STEP 2   ROUTE + AUTH
         routes/invoice.routes.js
         authShop  →  rejects if no valid shop token
         │
         ▼
STEP 3   IDEMPOTENCY CHECK
         controllers/invoiceController.js  generate()
         Invoice.findOne({ orderId })
         already exists?  →  return it, stop here
         │
         ▼
STEP 4   LOAD DATA
         controllers/invoiceController.js  loadInvoiceInput()
         │
         ├─ Order.findById(orderId)              6aa8e275…
         ├─ ShopAuth.findById(order.shopId)      6aa8dda0…
         ├─ stateCodeFromAddress(shop.address)   → "29"   supplier state
         ├─ Customer.findById(order.customerId)  Kajal, Maharashtra
         └─ Product.find() + ShopProductStatus.find()
                                                  → HSN, MRP, price
         │
         ▼
STEP 5   DECIDE CGST/SGST vs IGST
         deliveryType === "SELF_PICKUP"  →  place of supply = shop state "29"
         deliveryType === "HOME_DELIVERY" →  place of supply = buyer state
         │
         placeOfSupply "29" === supplier "29"   →  INTRA-STATE → CGST + SGST
         placeOfSupply "27" !== supplier "29"   →  INTER-STATE → IGST
         │
         ▼
STEP 6   BUILD THE INVOICE  (no database here)
         services/invoiceService.js  buildInvoice()
         │
         ├─ buildLines()                   one line per order item
         │    │
         │    ├─ resolveClassification()   which GST rate?
         │    │     └─ config/gstRates.js  resolveHsnRate()  longest-prefix
         │    │          KitKat    180690  → 18%
         │    │          Britannia 19053100 → 18%
         │    │     no HSN match → customGstRate → none → BLOCKED
         │    │
         │    └─ calculateTaxLine()        services/gstCalculator.js
         │         gross     = price × qty            (GST-inclusive)
         │         taxable   = gross / 1.18
         │         cgst = sgst = taxable × 0.09
         │         lineTotal = taxable + cgst + sgst
         │
         ├─ calculateInvoiceTotals()      sum all lines, compute roundOff
         ├─ convertRupeesToIndianWords()  "Forty Rupees Only"
         ├─ formatInvoiceNumber()         Rule 46  →  INV-2627-00001
         └─ isValidGstin()                15 chars?  →  no → leave blank
         │
         ▼
STEP 7   SAVE
         Invoice.create({ ...invoice, merchantId })
         models/ShopModels/invoice.model.js
         │
         ▼
STEP 8   RESPOND  201 Created
```

## B. Rate lookup (no order, no invoice)

```
GET /api/gst/hsn-rate/180690?price=40&quantity=1
   │
   ▼
routes/ShopRoutes/shopGst.routes.js
   │
   ▼
services/gstCalculator.js
   resolveClassification()  →  config/gstRates.js  →  18%
   calculateTaxLine()       →  33.90 + 3.05 + 3.05 = 40
   │
   ▼
200 OK
```

## C. Single product preview (shop app)

```
GET /api/shopProduct/gst-preview/:productId?placeOfSupplyCode=29
   │
   ▼
shopProduct.routes.js  →  authShop
   │
   ▼
shopProduct.controller.js  getGstPreview
   │
   ▼
shopProduct.service.js  getGstPreviewService
   ├─ reads this shop's own HSN for that product
   ├─ config/states.js  →  supplier state from shop address (not hardcoded)
   └─ services/gstCalculator.js  calculateTaxLine()
   │
   ▼
200 OK
```

## D. Where the numbers come from

```
                    ┌──────────────────────┐
                    │  config/gstRates.js  │  WHAT the rate is
                    │  78 HSN entries            │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │ services/            │  HOW the tax is
                    │ gstCalculator.js     │  calculated
                    │ paise maths          │
                    └──────────┬───────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
   rate lookup           product preview          tax invoice
   (path B)              (path C)                (path A)
```

One engine, three callers. The preview and the invoice cannot disagree, because both call `calculateTaxLine()`.

---

## E. Both products, side by side

```
                    KITKAT                    BRITANNIA
productId      6aa6e6265bb3bcc1bcbf3344    6aa8e43bfaa40450b50da4e1
customHsnCode  180690                      19053100
lookup         exact match                 exact match
gstRate        18%  (from table)           18%  (from table)
customGstRate  undefined                   undefined
price          40                          75
MRP            75                          110
                    │                            │
                    ▼                            ▼
              gross = 40×1 = 40           gross = 75×1 = 75
              taxable = 40/1.18           taxable = 75/1.18
                     = 33.90                     = 63.56
              cgst = 33.90×0.09           cgst = 63.56×0.09
                   = 3.05                       = 5.72
              sgst = 3.05                  sgst = 5.72
                    │                            │
                    ▼                            ▼
              total = 40.00                total = 75.00
```

## F. Persisted invoice

Only one exists in the database.

```
POST /api/invoice/generate  { orderId: "6aa8e275faa40450b50da051" }
                    │
                    ▼  order is SELF_PICKUP, paymentStatus Success
              saved to MongoDB
                    │
                    ▼
invoiceNumber    INV-2627-00001        FY 2627, Rule 46, ≤16 chars
orderId          6aa8e275faa40450b50da051
merchantId       6aa8dda0faa40450b50d917b
documentType     BILL_OF_SUPPLY
placeOfSupply    29  Karnataka
isInterState     false
taxSplit         CGST + SGST
seller.gstin     ""                   gstinRaw  3627U46UFBN (invalid, 11 chars)

items[0]         KitKat Chocolate Bar
                 hsn 180690 · qty 1 · mrp 75 · unitPrice 40
                 taxable 33.90 · gst 18% · cgst 3.05 · sgst 3.05 · line 40.00

totals           gross 40 · discount 0 · taxable 33.90
                 cgst 3.05 · sgst 3.05 · igst 0 · cess 0 · roundOff 0
                 grandTotal 40.00
```

Calling `generate` again with the same `orderId` returns this same invoice — it does not create a second one.


**3. Only 2 of 51 products have an HSN.** The rest are blocked at invoicing rather than charged 0%.

