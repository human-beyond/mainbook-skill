# MainBook bank statements — an agent skill

An agent skill for turning PDF bank and credit-card statements into checked Excel, CSV or JSON — and, more importantly, for **checking the result before it is handed to anyone**.

Statement PDFs are one of the few documents where a plausible-looking answer is worse than no answer. A row read as `1,234.56` instead of `123.45` still looks like a transaction and still sums to something. This skill teaches an assistant the workflow that catches that: look before spending credits, convert, re-add the statement, and report flagged rows instead of smoothing them over.

The extraction and the arithmetic check are done by [MainBook](https://mainbook.ai/mcp) through its MCP server. This repository is the skill — the judgement around the tool, not a copy of it.

## Install

```bash
npx skills add human-beyond/mainbook-skill
```

Or copy `SKILL.md` and `references/` into your agent's skills directory.

The skill needs MainBook's MCP server to be reachable — either the published package (`uvx mainbook-mcp`) or the hosted endpoint at `https://mcp.mainbook.ai/mcp`. Both use your own MainBook API key from <https://mainbook.ai/developer>. Setup details are in [`references/setup.md`](references/setup.md).

## What it covers

- checking the page-credit balance and telling the person the cost **before** converting;
- picking the output format for what actually happens next — a human opening a spreadsheet, an import into accounting software, or answering questions in the conversation;
- reading the validation verdict honestly: opening balance + credits − debits against the printed closing balance;
- treating flagged rows as a human's job rather than noise to summarise away;
- merging several statements into one workbook while catching overlapping periods and missing months;
- credit cards, where a per-row running balance does not exist and its absence is not an error;
- the failure paths — password-protected PDFs, poor scans, the 50 MiB / 500 page limits, and jobs that outrun the polling window.

## What it will not do

MainBook reads statement files you already have. It does not connect to bank accounts, has no Open Banking access, and holds no banking credentials. There are no tools for buying credits, taking payments, deleting jobs, or changing account data — and the skill is explicit about saying so rather than improvising.

## Related

- [`human-beyond/mainbook-mcp`](https://github.com/human-beyond/mainbook-mcp) — the MCP server this skill drives
- [`ai.mainbook/bank-statement-converter`](https://registry.modelcontextprotocol.io/v0.1/servers?search=ai.mainbook) — its entry in the official MCP Registry
- [mainbook.ai/mcp](https://mainbook.ai/mcp) — product documentation

MIT licensed. Maintained by Human Beyond LLC.
