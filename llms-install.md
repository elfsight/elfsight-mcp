# Installing the Elfsight MCP server (for AI agents)

This file is a machine-readable install guide: an agent capable of editing config files or
running shell commands can follow it directly.

## 1. Sign-in

The server uses OAuth. The user signs in to Elfsight in a browser and picks the workspace to
grant; an agent cannot do that step for them. There is no API key to ask for.

## 2. Configure the connection

- Server URL: `https://mcp.elfsight.com/mcp`
- Transport: Streamable HTTP
- Auth: OAuth, discovered by the client from the server's `401` response. Do not add an
  `Authorization` header yourself and do not put credentials in the URL.

Add it to the MCP settings of the client you are running in, not to another editor's config
found in the workspace. Use whatever mechanism your host application exposes for a remote HTTP
MCP server (see `README.md` in this repository for the usual config shapes).

Cline: add it to `cline_mcp_settings.json` and keep `"type": "streamableHttp"`; without it Cline
uses the legacy SSE transport and the connection fails. Recent Cline versions keep the file in
`~/.cline/data/settings/`, older ones in the editor's
`User/globalStorage/saoudrizwan.claude-dev/settings/`. Do not create `.vscode/mcp.json` for
Cline: that file is VS Code's built-in MCP config and Cline does not read it.

```json
{
  "mcpServers": {
    "elfsight": {
      "type": "streamableHttp",
      "url": "https://mcp.elfsight.com/mcp"
    }
  }
}
```

Then ask the user to finish
the browser sign-in when the client prompts for it — in Claude Code that is `/mcp` →
**Authenticate**.

## 3. Verify the connection

Call the `list_account_apps` tool with no arguments. A working connection returns a JSON object
with an `apps` array (the apps installed in the authorized workspace, each with `alias` and
`name`). Report the result to the user, including
the app names, so they can confirm this is the right workspace.

## 4. Common errors

- **401 Unauthorized** — the user has not signed in yet, or the session expired. Ask them to
  authenticate again in the client; do not retry in a loop.
- **429 Too Many Requests** — the rate limit was exceeded. The response includes a
  `Retry-After` value in seconds; wait at least that long before retrying.
- **`list_account_apps` returns an empty `apps` array** — the workspace has no widgets yet. This is not a
  connection problem; use `list_app_catalog` and `list_widget_templates` to create the first
  widget.
- **A tool error with code `WIDGET_DOES_NOT_EXIST`** — the widget is not in the authorized
  workspace. Resolve `widgetPid` from `list_widgets`; the user may have granted a different
  workspace.
- **`create_widget` returns `WIDGET_PUBLISH_FAILED` with a `widgetPid`** — the widget was
  created but not published. Call `publish_widget` with that `widgetPid`; do not create
  another widget.

## 5. What not to do

- Do not guess or hardcode `widgetPid`, `templatePid`, `projectPid` or `appAlias` — resolve
  them from the list tools.
- Do not attempt to call any tool not listed in `tools/list`; the server exposes a fixed,
  versioned set.
- `create_widget`, `duplicate_widget`, `rename_widget`, `update_widget_settings`,
  `publish_widget` and `discard_widget_changes` change the user's widgets. Call them only for
  changes the user asked for. `publish_widget` and `discard_widget_changes` act on the whole
  draft, which may hold the user's own edits from the Dashboard, and publishing changes what
  visitors of the user's website see: get the user's explicit confirmation first.
- Do not open or fetch `publishUrl` from `get_widget_edit_links`: in a signed-in browser it
  publishes the draft immediately.
