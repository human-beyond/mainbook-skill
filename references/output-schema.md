# MainBook result schema

Read this before summarising a conversion, explaining a mismatch, or combining statements. The MCP JSON and the downloaded spreadsheet are different representations; do not treat the flattened spreadsheet columns as the JSON shape.

## MCP tool-result envelope

Both `convert_bank_statement` and `get_conversion` return a `ConversionOutput` with:

| Field | Type | Meaning |
|---|---|---|
| `job_id` | string | Conversion job identifier used for recovery and retrieval |
| `state` | string or null | Current job state |
| `pages` | integer or null | PDF pages recorded for the job |
| `validation` | object or null | Small statement-math verdict described below |
| `result_type` | `json`, `xlsx`, or `csv` | Requested representation |
| `data` | object or null | Present for a successful JSON result |
| `saved_file` | object or null | Local stdio path and placement reason for a written result |
| `download` | object or null | Hosted XLSX/CSV REST endpoint and retrieval instruction |
| `timed_out` | boolean | Whether the call stopped polling while the job continued |
| `message` | string | Actionable status text |

For a successful XLSX or CSV call, transaction rows are not embedded in this envelope. Retrieve the same `job_id` with `result_type="json"` for evidence; that retrieval spends no page credits.

## JSON: `data`

Successful JSON is nested exactly as:

```text
data
├── document
├── transactions[]
└── has_warnings
```

### `data.document`

| Fields | Type |
|---|---|
| `id`, `display_name`, `bank_name`, `account_holder`, `account_address`, `account_number_masked`, `account_type`, `kind`, `currency`, `period_start`, `period_end` | string |
| `billing_cycle_length_days` | integer or null |
| `starting_balance_cents`, `ending_balance_cents`, `net_credits_cents`, `net_debits_cents` | integer cents |
| `transactions_count` | non-negative integer |
| `credit_limit_cents`, `available_credit_cents`, `previous_balance_cents`, `new_balance_cents`, `payment_due_amount_cents` | integer cents or null |
| `payment_due_date` | string or null |
| `pages` | positive integer or null |

All `*_cents` values are integer minor units. Divide by 100 only for display; preserve the integers for arithmetic.

### `data.transactions[]`

| Field | Type / allowed values | Meaning |
|---|---|---|
| `row` | positive integer | Export row number |
| `source_file` | string | Source statement filename |
| `page` | positive integer | PDF page |
| `line_index` | non-negative integer | Source line index on that page |
| `date` | string | Returned transaction date |
| `description` | string | Returned narrative |
| `amount_cents` | integer cents | Signed transaction amount |
| `transaction_type` | `credit` or `debit` | Returned direction |
| `balance_after_cents` | integer cents or null | Running balance when present |
| `currency` | string or null | Row currency when present |
| `validation_status` | `valid`, `system_mismatch`, `user_edited_mismatch`, `user_reviewed`, or `user_accepted` | Row-level validation state |
| `warning_flags` | array of strings | Exact returned warning codes/messages to quote, not diagnose |
| `cardholder` | string | Cardholder section for applicable business-card rows; otherwise empty |
| `cardholder_card_masked` | string | Masked card for applicable business-card rows; otherwise empty |

`data.has_warnings` is a boolean summary. Still inspect every transaction's `warning_flags` and `validation_status`; do not rely on the summary alone.

## Validation object

| Field | Type | Meaning |
|---|---|---|
| `reconcilable` | boolean or null | Whether statement math can be reconciled; null can apply when the document has no running-balance reconciliation |
| `passed` | boolean | Whether MainBook's applicable extracted statement-math checks passed |
| `mismatched_rows` | non-negative integer | Rows that remain mathematically mismatched |

This proves only arithmetic consistency of the extracted statement model. It does not prove that every description, date, account field, or amount was extracted correctly, and offsetting extraction errors can still reconcile.

## XLSX and CSV columns (not MCP JSON)

The downloaded XLSX/CSV is a flattened row-oriented export. Its human-facing columns are separate from the nested JSON fields above.

Transaction columns are `Source File`, `Row`, `Page`, `Date`, `Description`, `Amount`, `Type`, `Balance`, and `Warning`. Statement/account columns may include `Document ID`, `Bank`, `Account Holder`, `Account Address`, `Account Number`, `Account Type`, `Currency`, `Statement Period`, `Billing Cycle Length`, `Starting Balance`, `Ending Balance`, `Net Credits`, `Net Debits`, and `Pages`. Card-cycle columns may include `Credit Limit`, `Available Credit`, `Previous Balance`, `New Balance`, `Payment Due Amount`, and `Payment Due Date`.

In spreadsheets, money is represented in currency units rather than integer cents, and the available columns can vary with the statement. Use the JSON evidence pass for counts, warnings, validation statuses, and arithmetic explanations instead of inferring them from the MCP XLSX/CSV tool result.
