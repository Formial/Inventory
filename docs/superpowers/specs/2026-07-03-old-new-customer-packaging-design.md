# Old vs New Customer Packaging — Design

## Problem

Each month has old (repeat) and new customers. Some packaging items — card,
brochure, glass bottle — go only to new customers as part of a "new customer
package." Old customers only ever get the plastic pump appropriate to their
kit size. Today packaging usage is derived purely from Rx count per kit
(20g/50g/foam), with no concept of new vs old customer, so new-only items
can't be tracked or costed correctly, and the report/PDFs don't separate them.

## Requirements

1. User manually enters **count of new customers** and **count of old
   customers** each report period (not derivable from Rx text).
2. Each packaging item gets a **Customer type**: `All` (today's behavior —
   usage = Rx count for its kit) or `New only` (usage = manual new-customer
   count, independent of kit).
3. Existing pump/box/sleeve/sticker items default to `All`. Card, Brochure,
   Glass bottle are new-only.
4. Manual packaging waste (existing Manual Usage → Packaging waste feature)
   must appear in **both** Summary and Detail PDFs, and in the on-screen
   Report, itemized per item (e.g. "Brochure — 3 wasted").
5. Close Month confirmation must show the new/old counts being used before
   committing stock deductions.

## Data model changes

- `pkg` items gain a field: `custType: 'all' | 'new'` (default `'all'` for
  existing rows via migration on load).
- New persisted values: `sv('newCust', n)`, `sv('oldCust', n)` — same
  pattern as existing `cprod`/`fprod`.

## Calculation changes (`calcAll`)

For each pkg item:
```
au = p.custType === 'new'
  ? newCust
  : (p.kit==='20g' ? n20 : p.kit==='50g' ? n50 : nf)
```
Everything downstream (cost, remaining, waste overlay) unchanged — waste
already applies per item name regardless of custType.

No change to ingredient deduction (API/CB/FB) — unaffected by customer type.

## UI changes

**Stock → Packaging table**: new column `Customer type` with a `<select>`
(All / New only) per row, same edit pattern as the existing `Kit` select.

**Report page**: two number inputs next to the date range — `New customers`,
`Old customers` — persisted via `sv`, read by `generateReport()` and
`calcAll()`. Changing them re-renders the report (same `onchange` pattern as
`cream-prod`).

**Packaging report table** (`t-pkg` + `dlDetail` + `dlSummary`): add a
`Customer type` column showing All/New; "Rx used" column logic already
described above — Waste column already exists on screen and in `dlDetail`'s
table needs adding (currently missing in the Detail PDF's packaging table,
confirmed via screenshot). Summary PDF gets a new small table: packaging
waste itemized by name + qty + cost, mirroring what's already shown for
manual API waste.

**Close month**: confirm dialog text includes the current New/Old counts
being applied, e.g. "New customers: 80, Old customers: 120". Deduction loop
uses the same `au` formula as `calcAll`.

## Out of scope

- No per-Rx tagging of individual prescriptions as new/old (counts are
  aggregate manual entry, not tied to specific daily-log lines).
- No validation that New+Old count equals total Rx count — they're
  independent manual inputs (a customer may have multiple Rx, or none this
  visit).
