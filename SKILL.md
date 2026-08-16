---
name: mainbook-bank-statement-converter
description: Turn PDF bank and credit-card statements into checked Excel, CSV or JSON, and verify the numbers before handing them over. Use this whenever someone has statement PDFs and wants transactions in a spreadsheet, needs statements prepared for bookkeeping, an accountant, a loan or tax filing, asks to "extract transactions", "convert my statements", "get this into Excel", or drops a folder of statements and asks what is in them — even if they never say the word MainBook. Also use it when a conversion has already run and the numbers need checking, or when several statements have to become one workbook.
---

# Bank statements into checked spreadsheets

Statement PDFs are one of the few documents where a plausible-looking answer is worse than no answer. A row read as `1,234.56` instead of `123.45` still looks like a transaction, still sums to something, and quietly poisons a tax return or a loan application months later. So the job is not "extract the table". The job is **extract, then prove the extraction is right, then say plainly what could not be proved.**

This skill covers that whole arc using MainBook, which does the extraction and the arithmetic check. Your part is choosing what to convert, reading the verdict honestly, and telling the person what they actually have.

## What you need

MainBook's MCP server exposes five tools: `convert_bank_statement`, `get_conversion`, `list_conversions`, `get_balance`, `output_folder`.

Check whether those tools are available before promising anything. If they are missing, say so and offer the setup in `references/setup.md` — do not fall back to reading the PDF yourself and typing numbers out of it. An assistant transcribing a statement by eye is exactly the failure mode this skill exists to prevent: it will look right and be wrong.

Conversion costs the account's page credits — one page, one credit. The other four tools are free to call. That single fact should shape the order you do things in.

## Before spending anything

1. **Look at what you were given.** File names and the first page tell you the bank, the period, and whether it is a bank account or a credit card. If someone points at a folder, list what is there and confirm the selection before converting — folders collect duplicates, and converting the same statement twice costs twice.
2. **Check the balance with `get_balance`.** It returns total, reserved and available credits, in pages. Compare that with the page count you are about to send. Finding out mid-job that the account is short wastes both the credits already spent and the person's time.
3. **Say what it will cost** when the job is more than a couple of pages: "these four statements are 38 pages, so about 38 credits". People are far more relaxed about spending when they were told first.

## Converting

Call `convert_bank_statement` with one source (`file_path` locally, or `file_url` for a public HTTPS link) and the format the person actually needs:

- **`xlsx`** when a human will open it. This is the default choice for "put it in a spreadsheet".
- **`csv`** when it feeds another system — accounting software, a script, an import wizard.
- **`json`** when you need the data in the conversation to answer questions about it. JSON comes back inline; XLSX and CSV are written to disk and you get the path.

Long statements can outrun the polling window. That is not a failure: the job keeps running on the server, and you get a `job_id`. Pick it up with `get_conversion` rather than starting a new conversion — a second conversion charges again for the same pages.

## Verifying — this is the part that matters

Every conversion comes back with a validation verdict, because MainBook re-adds the statement: opening balance plus credits minus debits has to equal the closing balance. Read it before you say anything about the result.

**When the arithmetic matches**, say so concretely — the numbers are the evidence: "63 transactions; opening 4,127.50 and closing 3,881.05 both match the statement, nothing flagged." That sentence tells someone they can use the file.

**When rows are flagged**, they are the story. MainBook flags what it could not read confidently instead of guessing, so a flag is a row a human has to look at — usually a smudged scan, an overlapping stamp, a handwritten correction. Name them, quote the line, and point at the page. Never summarise flags away as "a few minor issues".

**When the totals do not reconcile**, stop and say it plainly before anything else. Do not hand over a spreadsheet with a cheerful summary and the mismatch buried in a footnote. Give the difference, and the most likely reasons: a missing page in the PDF, a statement that starts mid-period, or a scan quality problem.

Credit-card statements are the honest exception. Card statements do not carry a running balance per row — only statement-level totals — so a per-row balance column is not missing data, it is data that does not exist. Explain that rather than treating it as an error, and check the card the way the statement itself does: opening balance, purchases, payments, closing balance.

## Several statements at once

Convert them one at a time — one job per statement keeps the flags attached to the statement that produced them.

When someone wants a single workbook, keep one sheet per statement and build the combined view on top, rather than pouring everything into one flat sheet. Then check two things people forget and later blame the tool for:

- **overlapping periods**, which silently double-count transactions in the overlap;
- **gaps between statements**, which look like a quiet month and are usually a missing PDF.

Both deserve a sentence in your summary even when nobody asked.

## When it does not work

- **Password-protected PDF** — MainBook cannot open it. Ask the person to save an unlocked copy; most banks' viewers can export one.
- **Photographed or low-quality scan** — expect flagged rows. Say up front that a re-scan will beat any amount of re-processing.
- **Over the limits** — 50 MiB or 500 pages per file. Split the file by statement period; splitting arbitrarily mid-statement breaks the balance check that makes the result trustworthy.
- **Nothing to convert** — if the PDF turns out to be a summary, a letter, or an account application rather than a statement, say that instead of converting it. It will cost credits and return almost nothing.

## What this cannot do, ever

MainBook reads statement files the person already has. It does not connect to bank accounts, has no Open Banking access, and holds no banking credentials. There are no tools for buying credits, taking payments, deleting jobs or changing account data.

If someone asks for a live balance or an account feed, tell them plainly that this is not that, and that anyone offering it needs their bank login — which is a completely different trust decision.

## Reference

- `references/setup.md` — connecting the MainBook MCP server (local package or hosted endpoint), where the API key comes from, and which client config to paste.
- `references/output-schema.md` — the columns a conversion returns, how dates and money are normalised, and what the validation fields mean.
