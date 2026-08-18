# Set up MainBook

Read this when the five MainBook tools are missing, authentication fails, or the person asks how credentials and client configuration work. Prefer local stdio for files on the person's machine.

## Sign in from a terminal

Run one command:

```bash
uvx mainbook-mcp auth login
```

The command opens `https://mainbook.ai/connect?code=...`, prints the same short code in the terminal, and waits for approval. Confirm that both codes match and approve in the browser; nothing is copied back. On first use, the approval page also asks the person to accept the Developer API Terms. The code is valid for 600 seconds.

The revocable credential is stored in the OS keyring when the optional `keyring` extra is installed and usable. Otherwise it is stored in `~/.config/mainbook/credentials.json`; the directory is mode `0700` and the file is mode `0600`. Check the state without exposing the credential:

```bash
uvx mainbook-mcp auth status
```

## Account and credits

A MainBook account is required. Creating a credential has no separate fee; conversions spend one page credit per PDF page. New accounts currently receive 20 page credits, which permits a small real test, but grants can change: `get_balance` is the live source for total, reserved, and available credits.

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

`MAINBOOK_API_KEY` remains available when an interactive browser cannot be used. It takes precedence over a stored terminal sign-in. Create and revoke manual keys at <https://mainbook.ai/developer>. Keep them in the process environment or a secret manager; never paste one into a conversation, commit one, print one, or place one in agent-created files.

## Hosted endpoint

`https://mcp.mainbook.ai/mcp` is unchanged. It authenticates only from that request's `Authorization: Bearer mb_live_...` header and ignores `MAINBOOK_API_KEY`, the OS keyring, and the local credential file. It has no OAuth flow.

Hosted mode cannot read local attachments, has no upload tool, cannot use `output_folder`, and rejects redirects. This is a hard stop for a local statement: never suggest publishing a bank statement to create a public URL. Use hosted mode only when the person already has a direct public HTTPS URL that returns the PDF without redirects.

For XLSX or CSV, hosted MCP returns an authenticated REST download endpoint rather than a local file. The client must fetch it with the same Bearer key. If the client cannot make that authenticated download, retrieve JSON inline or use local stdio.

## Sign out or rotate the stored credential

```bash
uvx mainbook-mcp auth logout
```

Logout attempts server-side revocation first, then removes the local stored copy; if server revocation fails, it warns the person to revoke at <https://mainbook.ai/developer>. Signing in again revokes the previous stored credential before saving the new one. `MAINBOOK_API_KEY`, when set, is separate and is not removed by logout.
