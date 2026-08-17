# MainBook skill terminal-login and plugin-bundle report

Report date: 2026-08-16 (America/New_York)

## Result

The skill is now an 83-line, state-driven workflow: tools missing or authentication failed → preflight → create or recover a job → retrieve evidence → deliver or combine results.

The repository is also a Claude Code marketplace and single-skill plugin. One local install exposes the root skill and registers a local stdio `mainbook` MCP server. The bundle contains no secret; terminal sign-in remains a separate human approval step.

No server source, production account, public registry, marketplace, catalogue, pull request, or remote branch was modified.

## Server-contract re-verification

The read-only source of truth was `mainbook-mcp/src/mainbook_mcp/`. These are the load-bearing claims from the skeptic report that I re-verified.

| Claim | Result | Source evidence |
|---|---|---|
| Exactly one PDF source is accepted and result types are JSON/XLSX/CSV. | Confirmed. | `models.py:52-117` |
| `convert_bank_statement` creates the paid page-credit job; the other four tools do not create one. | Confirmed. | `server.py:97-110`, `server.py:260-441` |
| Default polling is 50 seconds; timeout leaves the job running and returns a `job_id`. | Confirmed. | `models.py:11-15`, `models.py:81-90`, `server.py:231-258` |
| XLSX/CSV tool output does not embed transactions. JSON returns nested data; local binary results return `saved_file`; hosted binary results return `download`. | Confirmed. | `models.py:269-281`, `server.py:698-771` |
| The JSON evidence pass uses the same existing job and does not create another paid job. | Confirmed. `get_conversion` reads job/result state; only the conversion tool calls create/upload/start. | `server.py:330-386`, `server.py:698-730`, compared with `server.py:213-234` |
| JSON is nested under `data.document`, `data.transactions`, and `data.has_warnings`. | Confirmed. | `models.py:192-255` |
| JSON monetary fields are integer cents and rows include `line_index`, `validation_status`, `warning_flags`, `cardholder`, and `cardholder_card_masked`. | Confirmed. | `models.py:192-246` |
| Validation proves applicable extracted statement arithmetic, not complete extraction correctness. | Confirmed. The model exposes only `reconcilable`, `passed`, and `mismatched_rows`. | `models.py:159-170` |
| Warning causes must not be invented. | Confirmed. The contract returns flags and statuses, not physical causes such as stamps or handwriting. | `models.py:221-246` |
| There is no page-count-only MCP tool. | Confirmed. Page count is calculated inside PDF loading, while the server exposes only the five named tools. | `files.py:264-292`, `server.py:104-441` |
| Long XLSX/CSV recovery requires a destination because `get_conversion` cannot infer the original source folder. | Confirmed. | `server.py:525-609` |
| `idempotency_key` is supported and `list_conversions` provides a cursor-paginated recovery surface. | Confirmed. | `models.py:93-100`, `models.py:119-132`, `server.py:213-230`, `server.py:283-320` |
| Insufficient-credit errors can provide available and requested pages; no MCP tool buys credits. | Confirmed. | `errors.py:58-69`, `server.py:104-441`, `server.py:676-680` |
| MainBook has no workbook-merge tool. | Confirmed. None of the five tool definitions merges files. | `server.py:104-441` |
| Credit-card reconciliation and per-row balances are nullable rather than universally absent. | Confirmed. | `models.py:159-170`, `models.py:192-246` |
| Hosted mode cannot read local paths, cannot write local output, and uses a direct public HTTPS URL without redirects. | Confirmed. | `models.py:55-79`, `server.py:187-205`, `files.py:155-234` |
| Hosted authentication reads only the request Bearer header; stdio prefers `MAINBOOK_API_KEY`, then stored sign-in. | Confirmed. | `server.py:482-509` |
| HTTP 401 / invalid or revoked credentials have a specific re-login recovery. | Confirmed. | `errors.py:48-57` |
| Password-protected/unreadable PDFs fail validation before job creation; limits are 50 MiB and 500 pages. | Confirmed. | `files.py:22-23`, `files.py:264-292` |

### Claims from the skeptic report that are now outdated

The review was correct for the older release but its authentication-path conclusions no longer describe `mainbook-mcp` 0.5.0:

- "No integrated auth" and "no browser handoff" are now wrong. The CLI exposes `login`, `status`, and `logout`; login opens a browser-assisted device flow and never renders the resulting key (`__main__.py:75-92`, `__main__.py:224-263`, `auth.py:96-145`).
- Manual API-key handoff is no longer the interactive default. Stdio resolves `MAINBOOK_API_KEY` first for automation, then a stored credential (`server.py:496-509`).
- The old setup recommendation to have a newcomer create and copy a key was superseded by `uvx mainbook-mcp auth login`. Manual keys remain documented only for scripts, CI, and headless use.
- The report's statement that OAuth does not exist remains correct. The browser-assisted device flow produces an API key; it is not described as OAuth anywhere in the rewrite.
- The report's claim that skill installation alone does not configure MCP remains true for `npx skills add`, but the new Claude Code plugin path now bundles the skill and MCP configuration in one install.

Terminal credential storage is also source-confirmed: optional keyring support is declared in `pyproject.toml:43-46`; fallback path and preference order are in `credentials.py:87-159`; fallback directory/file permissions are enforced at `credentials.py:280-303`. Logout and replacement revocation are implemented at `__main__.py:187-263`.

## Plugin format verified on 2026-08-16

Primary sources:

- [Claude Code plugins reference](https://code.claude.com/docs/en/plugins-reference), live page last modified 2026-08-14.
- [Claude Code plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces), live page last modified 2026-08-13.
- [Claude Code MCP documentation](https://code.claude.com/docs/en/mcp).
- [Vercel skills CLI discovery source](https://raw.githubusercontent.com/vercel-labs/skills/main/src/skills.ts), repository commit `c6f69c631292444cc541ac6d91e2226b0ff247da` dated 2026-08-10.

What the current documentation and validator establish:

- Plugin metadata lives at `.claude-plugin/plugin.json`; `name` is required, and `version`, `description`, and `author` are documented fields.
- A single `SKILL.md` at the plugin root is auto-loaded when there is no `skills/` directory and no `skills` manifest field. This current rule makes a duplicate `skills/.../SKILL.md` unnecessary.
- The skills CLI also checks the repository root first and returns that skill before scanning subdirectories (`src/skills.ts:229-242`). Therefore `npx skills add human-beyond/mainbook-skill` and the Claude plugin loader can share the same root file.
- Marketplace metadata lives at `.claude-plugin/marketplace.json`; required top-level fields are `name`, `owner`, and `plugins`. Each plugin entry requires `name` and `source`. A relative source resolves from the marketplace root, so `"./"` targets this repository.
- Current official Claude documentation shows plugin `.mcp.json` as `{ "mcpServers": { ... } }`. This contradicts the brief's older statement that `.mcp.json` must be a bare server map. I followed the current official schema. Claude Code 2.1.233 `plugin validate . --strict`, installation, inventory, and MCP health checks all accepted the wrapper.
- Codex has no Claude Code plugin bundle format. The repository says to install the root skill, add the TOML MCP entry, and run terminal login; it does not invent a Codex plugin.

### JSON schema field audit

| File | Field check |
|---|---|
| `.claude-plugin/plugin.json` | `name` is kebab-case; `version` is semver `1.0.0`; `description` is a string; `author.name` is present. No component paths override default discovery. |
| `.claude-plugin/marketplace.json` | `name`, `owner.name`, and `plugins` are present; optional `description` and `owner.url` have documented types; the entry has required `name` and relative `source`, plus documented `description`, `category`, and `homepage`. |
| `.mcp.json` | Top-level `mcpServers` object contains one `mainbook` stdio server; `command` is `uvx`; `args` starts with `mainbook-mcp` and carries three folder arguments; `env` is absent. |

`jq empty` parsed all three JSON files successfully. `claude plugin validate . --strict` then checked the marketplace and its `"./"` plugin entry against the runtime schema and returned `✔ Validation passed`.

## Local installation and load check

There was no `claude` executable on `PATH`. I downloaded the official macOS arm64 Claude Code 2.1.233 package into `/tmp`, used a separate temporary `CLAUDE_CONFIG_DIR`, and did not modify the person's persistent Claude configuration.

Commands run from `mainbook-skill/` (temporary paths abbreviated):

```bash
claude --version
claude plugin validate . --strict
CLAUDE_CONFIG_DIR=/tmp/mainbook-claude-config \
  claude plugin marketplace add "$PWD" --scope user
CLAUDE_CONFIG_DIR=/tmp/mainbook-claude-config \
  claude plugin install mainbook-bank-statement-converter@human-beyond --scope user
CLAUDE_CONFIG_DIR=/tmp/mainbook-claude-config claude plugin list --json
CLAUDE_CONFIG_DIR=/tmp/mainbook-claude-config \
  claude plugin details mainbook-bank-statement-converter@human-beyond
CLAUDE_CONFIG_DIR=/tmp/mainbook-claude-config claude mcp list
uvx mainbook-mcp auth status
```

Real outputs:

```text
2.1.233 (Claude Code)
Validating marketplace manifest: .../.claude-plugin/marketplace.json
✔ Validation passed

✔ Successfully added marketplace: human-beyond (declared in user settings)
✔ Successfully installed plugin: mainbook-bank-statement-converter@human-beyond (scope: user)
```

`plugin list --json` reported:

```json
{
  "id": "mainbook-bank-statement-converter@human-beyond",
  "version": "1.0.0",
  "scope": "user",
  "enabled": true,
  "mcpServers": {
    "mainbook": {
      "command": "uvx",
      "args": ["mainbook-mcp", "${HOME}/Downloads", "${HOME}/Desktop", "${HOME}/Documents"]
    }
  }
}
```

`plugin details` reported:

```text
Component inventory
  Skills (1)  mainbook-bank-statement-converter
  Agents (0)
  Hooks (0)
  MCP servers (1)  mainbook
  LSP servers (0)
```

The installed cached `SKILL.md` and repository-root `SKILL.md` had the same SHA-256:

```text
4b5c1c354a9f265d97dfe353b7b91b8c2a2b004920db8295d64ed841192dab88
```

The first MCP health attempt failed because sandboxing blocked uv's default cache/tool directories outside the workspace. After setting temporary `UV_CACHE_DIR`, `UV_TOOL_DIR`, and `UV_PYTHON_INSTALL_DIR`, the same installed plugin reported:

```text
plugin:mainbook-bank-statement-converter:mainbook: uvx mainbook-mcp ... - ✔ Connected
```

The separately discovered project-root `.mcp.json` remained `Pending approval`, which is Claude Code's project MCP trust boundary; it is distinct from the installed plugin server above.

The published package then returned:

```text
Signed in: no
API base: https://api.mainbook.ai
Run 'mainbook-mcp auth login' to sign in.
```

No login approval or credential creation was performed.

## Ten-scenario reread

I re-read the finished `SKILL.md` as an agent limited to the five MainBook tools plus explicitly available host capabilities.

| # | Scenario | Result in finished skill |
|---:|---|---|
| 1 | Folder of PDFs to Excel | Executable: inspect scope/duplicates, use external page tooling when available, one job per PDF, retrieve JSON, and use a separate spreadsheet capability or deliver separate exports (`SKILL.md:27-34`, `SKILL.md:70`). |
| 2 | Balance mismatch | Executable: mandatory JSON evidence, calculate the returned arithmetic difference, lead with mismatch, invent no cause (`SKILL.md:48-60`). |
| 3 | Card with no row balance | Executable: inspect `kind`, `reconcilable`, and row values; use cycle totals only when this statement lacks balances (`SKILL.md:56`). |
| 4 | Missing tools | Complete loop: identify client, configure one path, terminal login, browser approval, reload, and free `get_balance` verification (`SKILL.md:14-23`). |
| 5 | 12 available / 38 required | Hard stop with available, required, shortfall, human web purchase, re-check, and optional useful subset (`SKILL.md:29-32`). |
| 6 | Timeout | Executable: preselect destination, retain `job_id`, and call `get_conversion` with the same type and destination; no second paid job (`SKILL.md:40-44`). |
| 7 | Twelve statements into one workbook | Capability boundary is explicit; provenance fields are named; separate exports are the honest fallback (`SKILL.md:64-70`). |
| 8 | Hosted endpoint plus local attachment | Hard stop: no disk, no upload, no redirects, and no suggestion to publish the statement (`SKILL.md:28`; `references/setup.md:94-100`). |
| 9 | Lost response after creation | Executable: retained idempotency key and `list_conversions` inspection before any new paid job (`SKILL.md:38-44`). |
| 10 | Password-protected PDF | Executable: request an unlocked readable copy; unsupported viewer speculation was removed (`SKILL.md:72-76`). |

## URL checks

Every external URL present in the repository after this report was requested live with HTTP GET on 2026-08-16.

| URL | Result |
|---|---|
| `https://github.com/human-beyond` | 200 |
| `https://github.com/human-beyond/mainbook-skill` | 200 |
| `https://github.com/human-beyond/mainbook-mcp` | 200 |
| `https://mainbook.ai/#pricing` | 200 |
| `https://mainbook.ai/connect?code=...` | 200 after redirect to the login page, as expected for a placeholder code |
| `https://mainbook.ai/developer` | 200 after redirect to the login page |
| `https://mainbook.ai/mcp` | 200; the live page states `20 pages` for the new-account signup grant |
| `https://mcp.mainbook.ai/mcp` | 200 headers/body received; the GET stream remained open and the 20-second curl check timed out after receiving the status, consistent with an MCP streaming endpoint |
| `https://code.claude.com/docs/en/plugins-reference` | 200 |
| `https://code.claude.com/docs/en/plugin-marketplaces` | 200 |
| `https://code.claude.com/docs/en/mcp` | 200 |
| `https://raw.githubusercontent.com/vercel-labs/skills/main/src/skills.ts` | 200 |
| `https://api.mainbook.ai` | 404 at the bare API root; this is the API base printed by the CLI, not a documentation page |

No `/support` link was added.

## File-by-file changes

- `SKILL.md`: replaced the essay with five operational states; narrowed verification; made JSON retrieval mandatory and free; added exact auth, credit, timeout, idempotency, hosted, merge, warning, and delivery behavior; removed the rejected rhetoric and unsupported causes.
- `references/setup.md`: made terminal browser sign-in the default; added no-secret configs for Claude Desktop, Claude Code, Cursor, and Codex; documented storage, manual-key precedence, hosted header-only behavior, account/Terms/credit facts, and logout/rotation.
- `references/output-schema.md`: separated the MCP envelope, nested JSON, validation object, and flattened spreadsheet columns; documented every field in `models.py`, including cent amounts and row warnings/statuses.
- `README.md`: disclosed instruction-only scope, MCP/account dependency, and page-credit spend before installation; added the one-bundle Claude plugin path and kept the instruction-only skills CLI path explicit.
- `.claude-plugin/plugin.json`: added the Claude plugin manifest.
- `.claude-plugin/marketplace.json`: added the same-repository marketplace entry with source `"./"`.
- `.mcp.json`: added the local stdio server using `uvx mainbook-mcp` and folder arguments, with no secret or `env` field.
- `LICENSE`: deliberately unchanged.
- `mainbook-mcp/`: read-only; deliberately unchanged.

I deliberately did not add a second `SKILL.md`, a hosted default, a Codex plugin manifest, an API key placeholder in local configs, telemetry-producing public `npx skills add`, a public install, a push, a merge, a pull request, or catalogue submission.

## Unverified items

- A real person's browser approval, first-use Developer API Terms screen, keyring write, logout, rotation, and expired/revoked-credential recovery were not exercised because they create or mutate a credential.
- The 20-page grant was verified on the live public page, not by creating a new account; `get_balance` remains the live per-account source.
- No paid statement conversion was run. Production extraction quality, actual warning strings, and a binary spreadsheet payload were not live-tested.
- Hosted authenticated downloads were not tested because they require a manual key.
- The person's persistent Claude Code UI was not modified or restarted. The plugin was installed and loaded with the official 2.1.233 CLI in an isolated temporary config.
- A remote GitHub marketplace install was not tested because the branch was not pushed. Local-source marketplace add/install was tested.
- The public `npx skills add human-beyond/mainbook-skill` command was not run, per the no-telemetry constraint. Root-skill discovery was verified against the current skills CLI source instead.

## Branch, commits, and paid-API accounting

- Branch: `feature/terminal-login-rewrite`
- Starting commit: `8f9acb8a9c6cf8a723b6702ce2dad1997815b230`
- Implementation commit: `46b2e73cdb1b06eefe97d456a43e020eb0ea3839`
- Verification/report commit: `d68f00194fd50b94e41b828d16da5c6c9321f94e`
- The current HEAD after recording the preceding report hash is listed in the reviewer handoff; a commit cannot embed its own final hash.

Paid API calls: **0**. DataForSEO: 0; Exa: 0; Firecrawl API: 0; ScrapeCreators: 0; Apify: 0; MainBook conversion API: 0. Approximate paid spend: **$0.00**. Research and validation used only public read-only HTTP GETs, local validators, package downloads, and local MCP initialization/tool discovery.
