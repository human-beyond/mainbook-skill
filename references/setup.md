# Connecting MainBook

Read this when the MainBook tools are not available in the session, or when someone asks how to set this up.

Two ways in. Both use the person's own MainBook API key, created at <https://mainbook.ai/developer>; keys start with `mb_live_`.

## Local package (stdio)

The server runs on the person's machine, so it can read statements straight from their folders and write results next to them. This is the right choice for desktop assistants.

```json
{
  "mcpServers": {
    "mainbook": {
      "command": "uvx",
      "args": ["mainbook-mcp", "~/Downloads", "~/Desktop", "~/Documents"],
      "env": { "MAINBOOK_API_KEY": "mb_live_…" }
    }
  }
}
```

The folder arguments are the only places the server may read a statement from or write a result to — anything outside them is refused. `uvx` comes with `uv` (`brew install uv`, or the installer at <https://astral.sh/uv>). Python 3.11 or newer.

Codex reads TOML, so the same thing goes in `~/.codex/config.toml`:

```toml
[mcp_servers.mainbook]
command = "uvx"
args = ["mainbook-mcp", "~/Downloads", "~/Desktop", "~/Documents"]
[mcp_servers.mainbook.env]
MAINBOOK_API_KEY = "mb_live_…"
```

## Hosted endpoint (Streamable HTTP)

Nothing to install. Point a remote-capable client at `https://mcp.mainbook.ai/mcp` and send the key on every call:

```text
Authorization: Bearer mb_live_…
```

The trade-off is real and worth stating to the person: the hosted server has no access to their disk. Local paths and the output folder do not exist there, so a statement has to be reachable as a public HTTPS URL, and XLSX or CSV results come back as a download instruction rather than a file on their machine. For a folder of statements on a laptop, the local package is simply the better fit.

## Limits and cost

- One PDF per conversion, up to 50 MiB and 500 pages.
- Conversion spends page credits from the person's account: one page, one credit. `get_balance`, `list_conversions`, `get_conversion` and `output_folder` do not.
- Keys carry the whole account. They belong in the client configuration or the environment, never pasted into a document, a repository, or a chat message.

## If something is missing

- **No key** — send the person to <https://mainbook.ai/developer>; the key is shown once at creation.
- **Tools not appearing** — the client needs a restart after a configuration change; most only read MCP config at startup.
- **A conversion that "vanished"** — it is almost certainly still running. `list_conversions` shows recent jobs and `get_conversion` retrieves one by `job_id`. Starting again charges again.
