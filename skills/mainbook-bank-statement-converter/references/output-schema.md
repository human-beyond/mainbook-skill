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
| `download` | object or null | Hosted XLSX/CSV retrieval instruction, described below |
| `timed_out` | boolean | Whether the call stopped polling while the job continued |
| `message` | string | Actionable status text |

For a successful XLSX or CSV call, transaction rows are not embedded in this envelope. Retrieve the same `job_id` with `result_type="json"` for evidence; that retrieval spends no page credits.

## Where an XLSX or CSV file actually is

`saved_file` and `download` are mutually exclusive, and which one arrives depends on how MainBook is running.

| Field | Type | Meaning |
|---|---|---|
| `saved_file.path` | string | Local stdio only: absolute path of the written file |
| `saved_file.reason` | string | Why that destination was chosen |
| `download.job_id` | string | Job the instruction belongs to |
| `download.result_type` | `xlsx` or `csv` | Representation the instruction retrieves |
| `download.url` | string or null | One-time download link; needs no key or header |
| `download.expires_at` | string or null | When that link stops working |
| `download.rest_endpoint` | string or null | Fallback REST URL that requires a legacy `mb_live_` Bearer key |
| `download.instruction` | string | Human-readable retrieval text, including any failure reason |

Only `job_id`, `result_type`, and `instruction` are always present in `download`. The rest is a three-way branch:

- **`url` and `expires_at` set** — a browser-signed-in hosted caller. The link works once, expires ten minutes after it was issued, and needs no headers. Calling `get_conversion` again with the same `job_id` issues a fresh link for free.
- **`rest_endpoint` set instead** — either a hosted caller authenticated with a legacy `mb_live_` key, or a link that could not be issued. `instruction` says which. This endpoint only accepts an `mb_live_` Bearer key, so it is a dead end for a browser-signed-in person; retrieve JSON instead.
- **Neither, because `saved_file` is set** — local stdio wrote the file to disk.

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
| `amount_cents` | signed integer cents | Returned transaction amount, already signed: positive for a credit, negative for a debit |
| `transaction_type` | `credit` or `debit` | Returned direction, stated separately |
| `balance_after_cents` | integer cents or null | Running balance when present |
| `currency` | string or null | Row currency when present |
| `validation_status` | `valid`, `system_mismatch`, `user_edited_mismatch`, `user_reviewed`, or `user_accepted` | Row-level validation state; `user_accepted` means the account owner already cleared this warning, while `user_reviewed` means they looked and left it standing |
| `warning_flags` | array of strings | Exact returned warning codes/messages to quote, not diagnose |
| `cardholder` | string | Cardholder section for applicable business-card rows; otherwise empty |
| `cardholder_card_masked` | string | Masked card for applicable business-card rows; otherwise empty |

`data.has_warnings` is a boolean summary. Still inspect every transaction's `warning_flags` and `validation_status`; do not rely on the summary alone.

## Validation object

| Field | Type | Meaning |
|---|---|---|
| `reconcilable` | boolean or null | Whether statement math can be reconciled; null can apply when the document has no running-balance reconciliation |
| `passed` | boolean | Whether the job finished in the clean state, **not** an arithmetic verdict — see below |
| `mismatched_rows` | non-negative integer | Rows that remain mathematically mismatched |

`passed` is true only when the job ended in state `succeeded`. A job that ended `succeeded_with_warnings` returns `passed: false` **even when `mismatched_rows` is 0 and nothing is arithmetically wrong**. Reporting that as a failed arithmetic check is a false alarm. The arithmetic verdict is `mismatched_rows` read together with `reconcilable`; `passed` only says whether the job finished without raising any warning at all.

Even a clean verdict proves only arithmetic consistency of the extracted statement model. It does not prove that every description, date, account field, or amount was extracted correctly, and offsetting extraction errors can still reconcile.

## `get_balance`

| Field | Type | Meaning |
|---|---|---|
| `balance` | integer | Total page credits on the account |
| `reserved` | non-negative integer | Credits held by jobs already in flight |
| `available` | integer | Credits a new conversion can actually use |
| `units` | always `pages` | Unit for every number above |
| `explanation` | string | Returned wording to reuse when explaining the balance |

Compare required pages against `available`, not `balance`.

## `list_conversions`

Returns `conversions`, `next_cursor` (string or null), `count`, and `units`. Each entry carries `job_id`, `state`, `filename`, `file_format` (always `pdf`), `pages`, `credits_reserved` (integer or null), `source` (`web` or `api`), `created_at`, `updated_at`, `validation` (the object above, or null), and `error` with `category` and `reason` (each string or null).

`state` is one of `awaiting_upload`, `queued`, `processing`, `succeeded`, `succeeded_with_warnings`, `insufficient_credits`, `failed`, or `expired`. Use `filename`, `created_at`, `pages`, and `state` to recognise a job that already exists before creating a paid duplicate.

## XLSX and CSV columns (not MCP JSON)

The downloaded XLSX/CSV is a flattened row-oriented export. Its human-facing columns are separate from the nested JSON fields above.

Transaction columns are `Source File`, `Row`, `Page`, `Date`, `Description`, `Amount`, `Type`, `Balance`, `Warning`, and, for business-card statements, `Cardholder` and `Card`. Statement/account columns may include `Document ID`, `Bank`, `Account Holder`, `Account Address`, `Account Number`, `Account Type`, `Currency`, `Statement Period`, `Billing Cycle Length`, `Starting Balance`, `Ending Balance`, `Net Credits`, `Net Debits`, and `Pages`. Card-cycle columns may include `Credit Limit`, `Available Credit`, `Previous Balance`, `New Balance`, `Payment Due Amount`, and `Payment Due Date`.

In spreadsheets, money is represented in currency units rather than integer cents, and a column that would be empty for every row of this statement is dropped from the export entirely, so the column set varies between statements. Use the JSON evidence pass for counts, warnings, validation statuses, and arithmetic explanations instead of inferring them from the MCP XLSX/CSV tool result.
