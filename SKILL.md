---
name: mainbook-bank-statement-converter
description: Use when a user wants PDF bank or credit-card statements converted through the MainBook MCP tools to XLSX, CSV, or JSON, or wants an existing conversion checked for arithmetic mismatches and warning rows. Covers page-credit preflight, timeouts, local versus hosted file handling, and multi-statement workflows.
---

# Convert bank statements and report the evidence

Use MainBook to convert statement PDFs and report what its arithmetic validation and row-level flags actually show. A passed check means the extracted statement math reconciles; it does not prove that every field was extracted correctly.

## State 1: tools missing or authentication failed

MainBook exposes five tools: `convert_bank_statement`, `get_conversion`, `list_conversions`, `get_balance`, and `output_folder`. Only `convert_bank_statement` creates a paid page-credit job; the other calls do not spend page credits.

If the tools are missing:

1. Identify the MCP client and read `references/setup.md`.
2. Walk the person through only the applicable local-stdio setup path.
3. Ask them to run `uvx mainbook-mcp auth login`, confirm the matching browser code, and approve sign-in. Never ask them to paste a key or token into the conversation.
4. Ask them to reload the client, then verify with the free `get_balance` call. Never use a paid conversion as the setup test.

For HTTP 401 or another authentication error, ask the person to run `mainbook-mcp auth login` again and approve it in the browser. Wait for them, then retry one free read-only call before anything paid. Never print a credential or write one into a file you create.

Do not transcribe statement rows manually as a fallback.

## State 2: preflight before spending credits

1. Inspect the selected PDFs and resolve only real ambiguity: duplicates, unclear scope, or files that are not statements. One conversion accepts exactly one `file_path` or `file_url`.
2. For a local attachment, require local stdio. Hosted mode has no disk access or upload tool, and redirects are rejected. Stop rather than suggesting that a bank statement be published to obtain a URL.
3. Determine page counts with separate PDF/file tooling when available. MainBook has no page-count-only MCP tool. If no such tooling is available, say an exact credit estimate is unavailable; do not invent one. One PDF page spends one page credit.
4. Call `get_balance` and compare `available` with the known required pages. Include already `reserved` credits in the explanation when relevant.

If credits are insufficient, stop and state available, required, and shortfall. The person must add page credits in the web app because no MCP tool can. Re-check `get_balance` afterwards. Offer a smaller user-selected subset only when partial output is genuinely useful.

Choose `xlsx` for a person opening a workbook, `csv` for an import or script, and `json` for inline analysis. For XLSX or CSV, choose an allowed destination before starting.

## State 3: create or recover a job

Set and retain a unique `idempotency_key` for each paid request. Reuse it only when retrying the identical PDF request; never reuse it for a different file or options.

For a long local XLSX or CSV job, pass an explicit `output_path` or set a valid default with `output_folder` before calling `convert_bank_statement`. Keep the returned `job_id`, requested result type, and destination.

If polling times out, the job continues. Call `get_conversion` with the same `job_id`, result type, and output destination. Never start a second paid conversion for the same PDF.

After a lost response or any uncertain failure, inspect `list_conversions` before creating another job. Match on filename, timestamps, pages, and state; follow `next_cursor` when the first page is insufficient. If an existing job is still running, retrieve it with `get_conversion` after the suggested delay.

## State 4: retrieve the evidence

For every XLSX or CSV job, call `get_conversion` again with the same `job_id` and `result_type="json"` before summarising counts, balances, warning rows, or a mismatch. This mandatory evidence call spends no page credits. The XLSX/CSV tool result itself contains only a saved path or download instruction plus the small validation object, not the transactions.

Read the nested shape documented in `references/output-schema.md`: `data.document`, `data.transactions`, and `data.has_warnings`. Money fields such as `starting_balance_cents` and `amount_cents` are integer cents.

Report `validation.reconcilable`, `validation.passed`, and `validation.mismatched_rows`. Treat them as arithmetic evidence only. A passed result does not certify descriptions, dates, account fields, or every extracted amount.

For every row with non-empty `warning_flags` or `validation_status != "valid"`, quote the returned flags and status with source file, page, line index, date, description, and amount. Do not diagnose an unreturned physical cause. For a mismatch, calculate and show the arithmetic difference from the returned document totals without inventing a cause.

For cards, inspect `data.document.kind`, `validation.reconcilable`, and each `balance_after_cents`. If this statement has no per-row balance, explain that the issuer did not provide one and use the returned cycle totals; do not treat the missing field as an extraction error.

## State 5: deliver or combine results

Lead with a mismatch or unresolved warnings. Otherwise state that the extracted statement arithmetic reconciles, while preserving the distinction between reconciliation and extraction accuracy. Give the output path or retrieval limitation, transaction count, relevant balances, validation fields, and every flagged row.

Use this delivery checklist:

- Identify the source file, statement period, and `job_id`.
- Give the saved path or hosted retrieval limitation.
- State the transaction count and returned statement totals.
- State all three validation fields and any calculated difference.
- List every non-valid status and exact warning flag; say when there are none.

MainBook converts one PDF at a time and does not merge workbooks. For several statements, keep one job per PDF and retrieve JSON for every completed job. If a spreadsheet-writing capability is available, build one sheet per statement plus a combined view while preserving source file, page, line index, account, statement period, validation status, and warning flags. Check for overlapping periods and gaps. If no spreadsheet capability is available, deliver the separate exports and say the merge was not performed.

Handle terminal failures plainly:

- For a password-protected or unreadable PDF, ask for an unlocked readable copy.
- Enforce the 50 MiB and 500-page limits; split by complete statement periods, not mid-statement.
- If a file is not a statement, do not spend credits on it.

MainBook reads files the person already has. It does not connect to bank accounts, hold bank credentials, buy credits, take payments, delete jobs, or change account data.

## References

- `references/setup.md` for terminal sign-in, client configuration, hosted-mode boundaries, manual keys, and credential lifecycle.
- `references/output-schema.md` for the MCP JSON shape, tool-result envelope, and separate spreadsheet columns.
