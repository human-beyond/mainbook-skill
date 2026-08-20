---
name: mainbook-bank-statement-converter
description: Use when a user wants PDF bank or credit-card statements converted through the MainBook MCP tools to XLSX, CSV, or JSON, or wants an existing conversion checked for arithmetic mismatches and warning rows. Covers sign-in, page-credit preflight, timeouts, local versus hosted file handling, and multi-statement workflows.
---

# Convert bank statements and report the evidence

This file is instructions for an agent, not a converter. Converting anything requires the MainBook MCP server and a signed-in MainBook account, and each converted PDF page spends one page credit. Without those tools this skill cannot convert a statement and must not pretend otherwise.

Use MainBook to convert statement PDFs and report what its arithmetic validation and row-level flags actually show. A passed check means the extracted statement math reconciles; it does not prove that every field was extracted correctly.

## Two ways MainBook runs, and how to tell them apart

| | Local (stdio) | Hosted (`https://mcp.mainbook.ai/mcp`) |
|---|---|---|
| Tools offered | 5, including `output_folder` | 4; `output_folder` does not exist |
| PDF source | `file_path` on this machine, or `file_url` | `file_url` only — no disk, no upload tool |
| XLSX/CSV result | written to an allowed local folder | one-time download link the person opens |
| Sign-in | `mainbook-mcp auth login` in a terminal | browser sign-in inside the MCP client |

Read the tool list before deciding: if `output_folder` is present, this is local mode. Both modes spend the same page credits from the same account.

## State 1: tools missing or sign-in failed

MainBook exposes `convert_bank_statement`, `get_conversion`, `list_conversions`, `get_balance`, and — locally only — `output_folder`. Only `convert_bank_statement` creates a paid page-credit job; every other call is free.

If the tools are missing:

1. Identify the MCP client and read `references/setup.md`.
2. Walk the person through only the applicable path: local stdio for files on their machine, hosted for a client with no disk access.
3. For local, ask them to run `uvx mainbook-mcp auth login`, confirm the matching browser code, and approve. For hosted, ask them to add `https://mcp.mainbook.ai/mcp` in their client and complete the browser sign-in and approval screen.
4. Ask them to reload the client, then verify with the free `get_balance` call. Never use a paid conversion as the setup test.

Never ask the person to paste a key, token, or password into the conversation, and never print one or write one into a file you create.

Authentication failures have two different fixes, so name the mode before answering:

- **HTTP 401, `invalid_token`, or a message about an invalid or revoked credential.** Local: ask them to run `mainbook-mcp auth login` again and approve it in the browser. Hosted: ask them to reconnect MainBook in their client and complete the sign-in screen again. Either way, wait for them to confirm, then retry one free read-only call such as `get_balance` before anything paid.
- **HTTP 403 with `insufficient_scope`.** This connection never asked for the conversion permission, so approving cannot be retried into existence and neither can the paid call. The client must request `mainbook:convert` as well as `mainbook:read` — in Cursor that is the `scopes` list in the server entry — and the person must then reconnect and approve. Read calls keep working meanwhile.
- **A 403 naming unaccepted API terms.** Different problem, different fix: the account has not accepted the Developer API Terms. Send the person to <https://mainbook.ai/developer> to accept them. Changing scopes or reconnecting will not help.

Do not transcribe statement rows manually as a fallback.

## State 2: preflight before spending credits

1. Inspect the selected PDFs and resolve only real ambiguity: duplicates, unclear scope, or files that are not statements. One conversion accepts exactly one `file_path` or `file_url`.
2. For a local attachment, require local stdio. Hosted mode has no disk access and no upload tool, and rejects redirects. Stop rather than suggesting that a bank statement be published to obtain a URL.
3. Determine page counts with separate PDF/file tooling when available. MainBook has no page-count-only MCP tool. If no such tooling is available, say an exact credit estimate is unavailable; do not invent one. One PDF page spends one page credit.
4. Call `get_balance` and compare `available` with the known required pages. Include already `reserved` credits in the explanation when relevant.

If credits are insufficient, stop and state available, required, and shortfall. The person must add page credits in the web app because no MCP tool can. Re-check `get_balance` afterwards. Offer a smaller user-selected subset only when partial output is genuinely useful.

Choose `xlsx` for a person opening a workbook, `csv` for an import or script, and `json` for inline analysis. In local mode, choose an allowed destination before starting an XLSX or CSV job.

## State 3: create or recover a job

Set and retain a unique `idempotency_key` for each paid request. Reuse it only when retrying the identical PDF request; never reuse it for a different file or options.

For a long local XLSX or CSV job, pass an explicit `output_path` or set a valid default with `output_folder` before calling `convert_bank_statement`. `file_path` and `output_path` are advertised in hosted mode too, but hosted rejects both at call time, so pass neither there. Keep the returned `job_id`, requested result type, and destination.

`timeout_seconds` accepts 30 to 900 and defaults to 50, deliberately under the 60-second limit most MCP clients enforce. Raise it only for a client known to wait longer; a client that gives up first throws away the `job_id` and the conversion looks lost.

If polling times out, the job continues. Call `get_conversion` with the same `job_id`, result type, and output destination. Locally that destination is required — unlike the conversion call, `get_conversion` cannot guess a folder and will fail without `output_path` or a default set by `output_folder`. Never start a second paid conversion for the same PDF.

An account runs at most six conversions at once; a seventh returns a rate-limit error with a retry delay rather than starting. When converting many statements, keep at most six in flight and wait rather than treating the refusal as a failure.

Reusing an `idempotency_key` with a different file or different options returns a conflict, and keys never expire. Generate a fresh one per distinct request.

After a lost response or any uncertain failure, inspect `list_conversions` before creating another job. Match on filename, timestamps, pages, and state; follow `next_cursor` when the first page is insufficient. If an existing job is still running, retrieve it with `get_conversion` after the suggested delay.

## State 4: retrieve the evidence

For every XLSX or CSV job, call `get_conversion` again with the same `job_id` and `result_type="json"` before summarising counts, balances, warning rows, or a mismatch. This mandatory evidence call spends no page credits. The XLSX/CSV tool result itself contains only a saved path or a download instruction plus the small validation object, not the transactions.

Read the nested shape documented in `references/output-schema.md`: `data.document`, `data.transactions`, and `data.has_warnings`. Money fields such as `starting_balance_cents` and `amount_cents` are integer cents. This evidence pass returns every transaction inline, so on a several-hundred-page statement expect a large result and summarise from it rather than re-reading it repeatedly.

Report `validation.reconcilable`, `validation.passed`, and `validation.mismatched_rows`, and read them correctly. `passed` is true only when the job ended in the clean state; a job that ended with warnings returns `passed: false` even when `mismatched_rows` is 0 and the arithmetic is fine. Never announce an arithmetic failure on the strength of `passed` alone — the arithmetic verdict is `mismatched_rows` together with `reconcilable`, and `passed: false` with zero mismatched rows means "finished with warnings", which the warning rows below will explain. Even a clean verdict does not certify descriptions, dates, account fields, or every extracted amount.

`amount_cents` already carries its sign: positive for a credit, negative for a debit. Do not apply `transaction_type` as a sign on top of it.

For every row with non-empty `warning_flags` or `validation_status != "valid"`, quote the returned flags and status with source file, page, line index, date, description, and amount. Say when a row is `user_accepted`, which means the account owner already cleared that warning, or `user_reviewed`, which means they looked and left it standing. Do not diagnose an unreturned physical cause. For a mismatch, calculate and show the arithmetic difference from the returned document totals without inventing a cause.

For cards, inspect `data.document.kind`, `validation.reconcilable`, and each `balance_after_cents`. If this statement has no per-row balance, explain that the issuer did not provide one and use the returned cycle totals; do not treat the missing field as an extraction error.

## State 5: deliver or combine results

Lead with a mismatch or unresolved warnings. Otherwise state that the extracted statement arithmetic reconciles, while preserving the distinction between reconciliation and extraction accuracy. Give the output path or download link, transaction count, relevant balances, validation fields, and every flagged row.

Handing over an XLSX or CSV file:

- **Local mode** returns `saved_file.path`. Give the full path, and note that an existing file is never overwritten — MainBook writes `statement (2).xlsx` beside `statement.xlsx` — so quote the path that came back rather than the one requested.
- **Hosted mode** returns `download.url` with `download.expires_at`. Give the person that link and say plainly that it opens once, needs no key or header, and stops working at the stated expiry or the moment it is first opened — including an interrupted download, which still consumes it. If they lose it, call `get_conversion` again with the same `job_id` for a fresh link; that call is free.
- **Hosted mode without a link** returns `download.rest_endpoint` instead, which only a legacy `mb_live_` key can fetch. Do not send a browser-signed-in person there. Say the file could not be handed over, quote the returned reason, and deliver the JSON evidence instead.

Use this delivery checklist:

- Identify the source file, statement period, and `job_id`.
- Give the saved path, the one-time link with its expiry, or the retrieval limitation.
- State the transaction count and returned statement totals.
- State all three validation fields and any calculated difference.
- List every non-valid status and exact warning flag; say when there are none.

MainBook converts one PDF at a time and does not merge workbooks. Any combined workbook is built by the agent, not by MainBook. For several statements, keep one job per PDF and retrieve JSON for every completed job. If a spreadsheet-writing capability is available, build one sheet per statement plus a combined view while preserving source file, page, line index, account, statement period, validation status, and warning flags. Check for overlapping periods and gaps. If no spreadsheet capability is available, deliver the separate exports and say the merge was not performed.

Handle terminal failures plainly:

- For a password-protected or unreadable PDF, ask for an unlocked readable copy.
- Enforce the 50 MiB and 500-page limits; split by complete statement periods, not mid-statement.
- If a file is not a statement, do not spend credits on it.

MainBook reads files the person already has. It does not connect to bank accounts, hold bank credentials, buy credits, take payments, delete jobs, or change account data.

## References

- `references/setup.md` for terminal sign-in, hosted browser sign-in, client configuration, manual keys, and credential lifecycle.
- `references/output-schema.md` for the MCP JSON shape, tool-result envelope, download instruction, and separate spreadsheet columns.
