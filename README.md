# MainBook Bank Statement Converter — agent skill

This repository contains agent instructions, not the converter. Converting a file requires the `mainbook-mcp` server and a MainBook account. Each conversion spends one page credit per PDF page; installing the skill and calling the other four MCP tools does not spend page credits.

The instructions require the agent to retrieve the JSON evidence behind XLSX/CSV results, report arithmetic mismatches and row-level warnings without treating reconciliation as proof of perfect extraction, and preserve provenance across multiple statements.

## Install for Claude Code

The Claude Code plugin bundles the skill and a local stdio MCP configuration with no secret:

```text
/plugin marketplace add human-beyond/mainbook-skill
/plugin install mainbook-bank-statement-converter@human-beyond
```

Then sign in separately and reload Claude Code:

```bash
uvx mainbook-mcp auth login
```

The command opens a browser approval page and stores a revocable credential locally; no key is copied into the plugin. Verify the reloaded connection with the free `get_balance` tool. See [`references/setup.md`](skills/mainbook-bank-statement-converter/references/setup.md) for the full flow and folder configuration.

## Install the skill only

For agents supported by the open skills CLI:

```bash
npx skills add human-beyond/mainbook-skill
```

This command installs the instructions only. `skills/mainbook-bank-statement-converter/SKILL.md` is the single source used by the skills CLI, the Claude Code plugin, and the GitHub Copilot plugin. Non-plugin clients still need the MCP configuration and terminal sign-in described in [`references/setup.md`](skills/mainbook-bank-statement-converter/references/setup.md).

Codex does not use the Claude Code plugin format. Install the skill, add the local MCP entry to `~/.codex/config.toml`, and run `uvx mainbook-mcp auth login`.

## Scope

The workflow covers page-credit preflight, duplicate-charge protection, timeout recovery, mandatory JSON evidence retrieval, statement and card validation, warning rows, insufficient credits, local-versus-hosted boundaries, and multi-statement delivery.

MainBook reads statement files the person already has. It does not connect to bank accounts, hold banking credentials, buy credits, take payments, delete jobs, or change account data.

## Related

- [`human-beyond/mainbook-mcp`](https://github.com/human-beyond/mainbook-mcp) — MCP server source
- [MainBook MCP setup](https://mainbook.ai/mcp) — account and product setup reference
- [`references/output-schema.md`](skills/mainbook-bank-statement-converter/references/output-schema.md) — MCP JSON and spreadsheet representations

MIT licensed. Maintained by Human Beyond LLC.
