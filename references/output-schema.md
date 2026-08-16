# What a conversion returns

Read this when you need to answer questions about the data, build a combined workbook, or explain to someone why a column they expected is missing.

## Shape

One row per transaction. Statement-level facts repeat on every row, so a merged export never loses track of which account a row came from.

Twenty-nine columns are possible; a file gets only the ones its statement actually fills. **Empty columns are dropped rather than shipped blank**, which is why two statements from different banks can produce different column sets — that is normal, not a bug.

## The columns

**Every row**

| # | Column | Meaning |
|---|---|---|
| 1 | Source File | which uploaded statement this row came from |
| 2 | Row | sequential row number across the export |
| 3 | Page | page of the statement the line was read from |
| 4 | Date | ISO `YYYY-MM-DD`, so `03/04` is never ambiguous |
| 5 | Description | the narrative exactly as the statement prints it |
| 6 | Amount | a real number, negative for debits — not text |
| 7 | Type | Credit or Debit, in a column you can filter |
| 8 | Balance | running balance after the transaction, when the statement prints one |
| 9 | Warning | present only on rows that need a second look; the row is shaded too |

**Statement and account facts** (10–23): Document ID, Bank, Account Holder, Account Address, Account Number (masked exactly as the statement masks it), Account Type, Currency, Statement Period, Billing Cycle Length, Starting Balance, Ending Balance, Net Credits, Net Debits, Pages.

**Credit-card cycle only** (24–29): Credit Limit, Available Credit, Previous Balance, New Balance, Payment Due Amount, Payment Due Date.

## Formats

- **Money** lands in real number cells with a two-decimal accounting format, so `SUM` works immediately and a minus sign marks debits.
- **Dates** are rewritten to ISO `YYYY-MM-DD`, so a spreadsheet sorts them chronologically instead of alphabetically.
- **CSV** is UTF-8 with a byte-order mark and RFC 4180 quoting — accented names survive and a comma inside a description does not shift the columns.

## The validation fields

The conversion carries the arithmetic check: opening balance plus credits minus debits against the printed closing balance, to the cent. Treat it as the verdict on whether the file can be trusted, and quote the numbers when you report it.

The `Warning` column is per row and means "MainBook could not read this line confidently and refused to guess". A warning is a human's job, not a defect to hide: point at the page and quote the line.

## Why cards have no Balance column

Credit-card statements do not print a running balance per transaction — only cycle totals. So on a card export the Balance column is absent, and that is the statement's design rather than missing data. Check a card the way the issuer does: previous balance, purchases, payments, new balance.
