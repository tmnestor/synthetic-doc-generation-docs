# Invoice vs Receipt: Semantics, Tax Status, and Benchmark Consequences

**Status:** analysis note. Claims about the generator are verified against
`/Users/tod/Desktop/Synthetic_Doc_Generation` at the paths cited inline; the conceptual and
tax-law sections are general domain knowledge and are marked where they are unverified against a
primary source.

## 1. The core distinction

An invoice and a receipt are not two names for the same document. They sit on opposite sides of a
payment event.

| | Invoice | Receipt |
|---|---|---|
| Purpose | A *request* for payment | *Proof* that payment occurred |
| Issued | Before payment | At, or after, payment |
| Accounting effect | Creates a receivable (seller) / payable (buyer) — an accrual event | Records a cash movement — a settlement event |
| Typical distinguishing fields | Invoice number, due date, payment terms, "amount due" | Payment method, "amount paid", tender and change |
| Buyer identity | Required — the seller must know who owes the money | Usually absent — over-the-counter sale |
| Lifecycle role | Opens an obligation | Discharges an obligation |

An invoice creates an obligation; a receipt extinguishes one. This is why a single transaction can
legitimately produce both documents in sequence, and why an invoice stamped **PAID** is
functionally doing a receipt's job.

## 2. The Australian overlap: both can be a tax invoice

The invoice/receipt split is a distinction of *document form and timing*, not of *tax document
class*. Under Australian GST rules, claiming an input tax credit requires a compliant tax invoice
for purchases above $82.50 including GST. A point-of-sale receipt that carries supplier name,
supplier ABN, date, a description of what was supplied, and the GST amount satisfies those
requirements and *is* a tax invoice.

The practical consequence for document understanding: the two categories genuinely overlap. Any
classifier trained to treat "receipt" and "tax invoice" as mutually exclusive is being taught a
distinction the tax law does not draw. (Threshold figure and required elements: unverified against
current published guidance — confirm before citing externally.)

## 3. How the generator models the distinction

The generator's schema separates the two types far more weakly than section 1 would suggest.

From `config/field_definitions.yml:16-43`, the receipt and invoice field subsets are **identical
except for two fields**:

| Field | receipt | invoice |
|---|---|---|
| `DOCUMENT_TYPE` | ✓ | ✓ |
| `SUPPLIER_NAME` | ✓ | ✓ |
| `BUSINESS_ABN` | ✓ | ✓ |
| `BUSINESS_ADDRESS` | ✓ | ✓ |
| `INVOICE_DATE` | ✓ | ✓ |
| `IS_GST_INCLUDED` | ✓ | ✓ |
| `GST_AMOUNT` | ✓ | ✓ |
| `TOTAL_AMOUNT` | ✓ | ✓ |
| `LINE_ITEM_DESCRIPTIONS` | ✓ | ✓ |
| `LINE_ITEM_QUANTITIES` | ✓ | ✓ |
| `LINE_ITEM_PRICES` | ✓ | ✓ |
| `LINE_ITEM_TOTAL_PRICES` | ✓ | ✓ |
| `PAYER_NAME` | — | ✓ |
| `PAYER_ADDRESS` | — | ✓ |

Note what is absent from **both** subsets: there is no invoice number, no payment due date, no
payment terms, and no payment method field anywhere in the schema. The receipt type also reuses
the field name `INVOICE_DATE` for its own document date.

### 3.1 Worked example — `CASE001`

Both documents belong to the same case and carry the same date, drawn from
`ground_truth/receipts.yml:1-16` and `ground_truth/invoices.yml:1-19`:

| Field | `CASE001` receipt (`receipt_fuel`) | `CASE001` invoice (`tax_invoice_mixed`) |
|---|---|---|
| `DOCUMENT_TYPE` | `RECEIPT` | `INVOICE` |
| `SUPPLIER_NAME` | Ravensdale Health Store | Prime Consulting Partners |
| `BUSINESS_ABN` | 79 104 332 181 | 57 773 872 148 |
| `INVOICE_DATE` | 07/07/2024 | 07/07/2024 |
| `IS_GST_INCLUDED` | `true` | `false` |
| `GST_AMOUNT` | 1.24 | 802.88 |
| `TOTAL_AMOUNT` | 13.60 | 8831.70 |
| `LINE_ITEM_DESCRIPTIONS` | `Dishwashing Liquid\|Bandaids 40pk` | `Tax return preparation\|Trademark registration\|Business strategy consulting` |
| `PAYER_NAME` | *(not in subset)* | Robin Wood |
| `PAYER_ADDRESS` | *(not in subset)* | 130 Todd St, Newtown NSW 2042 |

The only structural signal separating these two records is the presence of the payer block — and
`DOCUMENT_TYPE` itself, which is the label being predicted.

## 4. Two consequences for the benchmark

### 4.1 Type classification is under-tested

Because the field subsets are near-identical, a model that predicts `INVOICE` where the ground
truth says `RECEIPT` loses almost nothing on field-level extraction scoring: eleven of the twelve
receipt fields are still valid invoice fields. The error surfaces only in the `DOCUMENT_TYPE`
column, and — for a misread invoice — as two missing payer fields.

The benchmark therefore measures field extraction robustly but barely penalises document-type
confusion. That matters because the payer block is the *weakest* of the real-world
discriminators listed in section 1; the strong ones (due date, payment terms, payment method,
invoice number) are not modelled at all.

### 4.2 Transaction links do not distinguish settlement semantics

`ground_truth/transaction_links.yml` contains 110 links. Verified by parsing the file:

- 55 receipt links and 55 invoice links.
- All 110 have `match_status: FOUND`; difficulty splits easy 52 / medium 36 / hard 22, matching
  the figure recorded in the project documentation.
- **All 110 links — including all 55 invoice links — have `receipt_date` equal to `bank_date`.**
- The per-link schema uses the keys `receipt_date` and `receipt_total` for invoice entries as well
  as receipt entries; there is no invoice-specific naming.

The same-day settlement of every invoice is the substantive issue. A receipt→bank-statement link
means "this payment has already settled, and the receipt is the evidence". An invoice→bank-statement
link should mean "a claim was raised, and a payment settled it some days or weeks later" — under
ordinary trade terms, 14 to 30 days after the invoice date. In this dataset an invoice settles on
its issue date, exactly like a card purchase.

Compare the `CASE001` pair, both linked to `CASE001_cba_standard.png`
(`ground_truth/transaction_links.yml:1-23`):

| | Receipt link | Invoice link |
|---|---|---|
| Document date | 07/07/2024 | 07/07/2024 |
| Bank date | 07/07/2024 | 07/07/2024 |
| Amount | 13.60 (exact match) | 8831.70 (exact match) |
| `bank_description` | `VISA DEBIT PURCHASE SQ *RAVENSDALE Alexandria AU` | `EFT PAYMENT PRIME CONSULTING PARTNERS` |
| `match_difficulty` | medium | easy |

The only difference in linking evidence is how badly the merchant name is mangled in the bank
narration — the payment rail wording (`VISA DEBIT PURCHASE` vs `EFT PAYMENT`) is incidental, since
no field records payment method. Date and amount align exactly in both cases.

So the easy/medium/hard difficulty split is grading **fuzzy string matching on merchant names**,
not reconciliation under document semantics. A real reconciliation task requires tolerating a date
gap for invoices, partial or aggregated payments, and payment references keyed to an invoice
number. None of those appear.

## 5. If the distinction should be modelled properly

The lever is field-level, not renderer-level:

1. Add `INVOICE_NUMBER` and `PAYMENT_DUE_DATE` to the invoice subset.
2. Add `PAYMENT_METHOD` to the receipt subset.
3. Allow `bank_date` to fall after `receipt_date` for invoice links, and rename the link keys to
   something type-neutral (`document_date` / `document_total`).

This is not a cosmetic change. It alters the 46-column schema, so it propagates to:

- `GroundTruth_Export_Spec.md` §2 (the authoritative field table) and the CORD/DocILE mappings
  that enumerate fields;
- `GroundTruth_Export_Spec.md` §8.6 and its render `diagrams/01-export-scoring-pipeline.svg`, if
  column counts appear there;
- the research document's capability matrix, per the consistency rule in `CLAUDE.md`;
- the upstream generator: new fields need renderer support, and — for Tier 2 — bounding-box
  capture at draw time.

Whether the added realism justifies that churn is a scoping decision, and should be taken
deliberately rather than as a side effect of another change. Recording the limitation is the
minimum: as it stands, the benchmark tests extraction of invoice-shaped and receipt-shaped
documents, not the ability to tell a demand for payment from evidence of one.

## 6. Verification status

| Claim | Status |
|---|---|
| Receipt and invoice field subsets differ only by `PAYER_NAME`, `PAYER_ADDRESS` | Verified — `config/field_definitions.yml:16-43` |
| No invoice number, due date, payment terms, or payment method field in the schema | Verified — no such keys present in `config/field_definitions.yml` |
| `CASE001` field values as tabulated | Verified — `ground_truth/receipts.yml:1-16`, `ground_truth/invoices.yml:1-19` |
| 110 links; 55 receipt + 55 invoice; easy 52 / medium 36 / hard 22; all `FOUND` | Verified — parsed `ground_truth/transaction_links.yml` |
| All 110 links have document date equal to bank date | Verified — parsed `ground_truth/transaction_links.yml` |
| Link schema uses `receipt_date` / `receipt_total` keys for invoices | Verified — parsed `ground_truth/transaction_links.yml` |
| $82.50 GST-inclusive tax-invoice threshold and required elements | **Unverified** — confirm against current published guidance |
| 14–30 day trade payment terms as the typical invoice settlement gap | **Unverified** — general commercial convention, no source cited |
