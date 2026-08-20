# Set up MainBook

Read this when the MainBook tools are missing, sign-in fails, or the person asks how credentials and client configuration work.

There are two ways to run MainBook, and they are set up differently:

- **Local (stdio)** — the server runs on the person's machine and can read their PDFs. Sign-in is one terminal command. Prefer this whenever the statement is a file they already have.
- **Hosted (`https://mcp.mainbook.ai/mcp`)** — the server runs on MainBook's infrastructure. Sign-in happens in the browser, inside the MCP client. It has no access to local files.

A MainBook account is required either way, and both ways spend page credits from that same account.

## Local: sign in from a terminal

Run one command:

```bash
uvx mainbook-mcp auth login
```

The command opens `https://mainbook.ai/connect?code=...`, prints the same short code in the terminal, and waits for approval. Confirm that both codes match and approve in the browser; nothing is copied back. On first use, the approval page also asks the person to accept the Developer API Terms. The code is valid for 600 seconds.

The revocable credential is stored in the OS keyring when the optional `keyring` extra is installed and usable. Otherwise it is stored in `~/.config/mainbook/credentials.json`; the directory is mode `0700` and the file is mode `0600`. Check the state without exposing the credential:

```bash
uvx mainbook-mcp auth status
```

## Hosted: sign in inside the client

Point the client at `https://mcp.mainbook.ai/mcp` over Streamable HTTP. No key, header, or secret goes into the configuration. The first tool call triggers a browser sign-in on `mainbook.ai` and an approval screen; approving stores a revocable authorization for that client.

The approval screen asks for two separate permissions:

| Permission | Covers |
|---|---|
| Read | `get_balance`, `list_conversions`, `get_conversion` |
| Convert | `convert_bank_statement` — spends page credits |

The screen grants exactly what the client asked for; there is no partial approval. A client that requests read only leaves conversions failing with HTTP 403 `insufficient_scope`, and the fix is a client configuration that requests `mainbook:convert` as well, followed by reconnecting.

### Claude and other chat clients

Add `https://mcp.mainbook.ai/mcp` as a custom connector. The tool list is readable without signing in, so a setup wizard may detect authentication as "None" — change it to the option meaning the server asks for authentication, or the first conversion will fail. Then complete the browser sign-in and approval.

### Cursor

```json
{
  "mcpServers": {
    "mainbook": {
      "url": "https://mcp.mainbook.ai/mcp",
      "auth": {
        "CLIENT_ID": "mainbook-cursor",
        "scopes": ["mainbook:read", "mainbook:convert"]
      }
    }
  }
}
```

### MCP Inspector

Inspector tries to register itself on the fly, which MainBook does not offer. Enter the client id `mainbook-mcp-inspector` once in its authentication settings and leave the client secret empty.

### What hosted mode cannot do

Hosted mode cannot read local attachments, has no upload tool, does not offer the `output_folder` tool, and rejects redirects. The `file_path` and `output_path` arguments are still advertised in the tool schema but are refused at call time in hosted mode, so passing them only wastes a round trip. This is a hard stop for a statement that lives on the person's machine: never suggest publishing a bank statement to create a public URL. Use hosted mode only when the person already has a direct public HTTPS URL that returns the PDF without redirects.

For XLSX or CSV, hosted mode returns a one-time download link in `download.url` with `download.expires_at`. The link needs no key or header, works once, and expires ten minutes after it is issued. Calling `get_conversion` again with the same `job_id` issues a fresh link and spends no page credits. If MainBook could not issue a link, the response carries `download.rest_endpoint` instead — that endpoint only accepts a legacy `mb_live_` key, so retrieve JSON instead of sending a browser-signed-in person there.

## Account and credits

Creating a credential or connecting a client has no separate fee; conversions spend one page credit per PDF page. New accounts currently receive 20 page credits once, which permits a small real test, but grants can change: `get_balance` is the live source for total, reserved, and available credits.

No MCP tool can add or buy credits. When the balance is short, the person must add page credits in the web app at <https://mainbook.ai/#pricing>; call `get_balance` again before starting a conversion.

## Configure a local stdio client

The folder arguments are the only places the server may read local PDFs from or write results to. Adjust them to existing folders the person intends to use. No API key or `env` entry belongs in these interactive-client blocks.

### Claude Desktop

Add this server under `mcpServers` in `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "mainbook": {
      "command": "uvx",
      "args": ["mainbook-mcp", "~/Downloads", "~/Desktop", "~/Documents"]
    }
  }
}
```

### Claude Code

Use this project `.mcp.json` configuration:

```json
{
  "mcpServers": {
    "mainbook": {
      "command": "uvx",
      "args": ["mainbook-mcp", "~/Downloads", "~/Desktop", "~/Documents"]
    }
  }
}
```

The installable Claude Code plugin in this repository already supplies the same MCP configuration, so plugin users only need terminal sign-in and a reload.

### Cursor

Use this server entry in Cursor's `mcp.json`:

```json
{
  "mcpServers": {
    "mainbook": {
      "command": "uvx",
      "args": ["mainbook-mcp", "~/Downloads", "~/Desktop", "~/Documents"]
    }
  }
}
```

### Codex

Codex has no Claude Code plugin format. Add the MCP entry to `~/.codex/config.toml`, install the root skill separately, and sign in from the terminal:

```toml
[mcp_servers.mainbook]
command = "uvx"
args = ["mainbook-mcp", "~/Downloads", "~/Desktop", "~/Documents"]
```

After changing any client configuration, reload the client and verify the connection with the free `get_balance` call, not a conversion.

## Manual key for scripts, CI, and headless use

`MAINBOOK_API_KEY` remains available when an interactive browser cannot be used. Over stdio it takes precedence over a stored terminal sign-in. Hosted mode also still accepts a manual key sent as `Authorization: Bearer mb_live_...`, which is the right choice for a script but not for a person: a key-authenticated hosted caller gets a REST download endpoint instead of a one-time link. Create and revoke manual keys at <https://mainbook.ai/developer>. Keep them in the process environment or a secret manager; never paste one into a conversation, commit one, print one, or place one in agent-created files.

## Sign out, disconnect, or rotate

Local terminal sign-in:

```bash
uvx mainbook-mcp auth logout
```

Logout attempts server-side revocation first, then removes the local stored copy; if server revocation fails, it warns the person to revoke at <https://mainbook.ai/developer>. Signing in again revokes the previous stored credential before saving the new one. `MAINBOOK_API_KEY`, when set, is separate and is not removed by logout.

Hosted browser sign-in: every approved client is listed under "Connected apps" at <https://mainbook.ai/developer> with a Disconnect button. Disconnecting there ends that client's access; removing the connector in the client alone does not.
