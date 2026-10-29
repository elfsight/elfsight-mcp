# Elfsight — MCP Server

> This repository holds the documentation and the registry manifest (`server.json`) of the
> Elfsight MCP server. The server itself is hosted by Elfsight; there is nothing to
> build or run here. For help, contact [Elfsight Help Center](https://help.elfsight.com/).

A remote [Model Context Protocol](https://modelcontextprotocol.io) server over your
[Elfsight](https://elfsight.com/) workspace. Point an MCP-capable AI agent at it and it can
browse your apps, projects and widgets, create widgets from templates, change their settings,
publish them and hand you the install code — without you clicking through the Dashboard.

The server is workspace-scoped: when you sign in, you pick one Elfsight workspace, and the
agent can reach only the apps, projects and widgets of that workspace.

Server URL: `https://mcp.elfsight.com/mcp` (Streamable HTTP).

## Connect

Sign-in is OAuth. Add the server URL to your MCP client; the client opens a browser window,
you sign in to Elfsight, choose the workspace and approve access. There is no key to copy.

### Claude (claude.ai, Desktop, mobile)

In Claude, open **Settings → Connectors**, choose **Add custom connector** and enter the server
URL `https://mcp.elfsight.com/mcp`, then sign in to Elfsight and approve access. On Team and
Enterprise plans an organization Owner may need to add the connector first.

Claude lists nine tools as read-only and six as write tools; on the connector's page you
choose for each group whether Claude asks before calling it.

### Claude Code

```bash
claude mcp add --transport http elfsight https://mcp.elfsight.com/mcp
```

Then run `/mcp` inside Claude Code and choose **Authenticate** for `elfsight`.

### Cursor

This repository is also a [Cursor plugin](https://cursor.com/docs/plugins/building): the manifest
is in [`.cursor-plugin/plugin.json`](./.cursor-plugin/plugin.json), the server connection in
[`mcp.json`](./mcp.json), and a rule with usage notes for the agent in [`rules/`](./rules).

To connect without the plugin, add the server to `.cursor/mcp.json` and authenticate it when
Cursor prompts:

```json
{
  "mcpServers": {
    "elfsight": {
      "url": "https://mcp.elfsight.com/mcp"
    }
  }
}
```

### Cline

In Cline, open **MCP Servers → Remote Servers** and add:

- Server Name: `elfsight`
- Server URL: `https://mcp.elfsight.com/mcp`
- Transport: **Streamable HTTP**

Then choose **Authenticate** for the server and sign in to Elfsight in the browser.

The same in Cline's own settings file, `cline_mcp_settings.json`. Recent versions keep it in
`~/.cline/data/settings/`, older ones in the editor's
`User/globalStorage/saoudrizwan.claude-dev/settings/`. Keep `type`: without it Cline falls
back to the legacy SSE transport. `.vscode/mcp.json` belongs to VS Code's built-in MCP
support; Cline does not read it.

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

### Other clients

Any client that supports remote MCP servers with OAuth (dynamic client registration and
PKCE) should connect with the server URL alone.

VS Code: [![Install in VS Code](https://img.shields.io/badge/VS_Code-Install_Server-0098FF?logo=visualstudiocode&logoColor=white)](https://vscode.dev/redirect/mcp/install?name=elfsight&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Fmcp.elfsight.com%2Fmcp%22%7D)
or add it to `.vscode/mcp.json`:

```json
{
  "servers": {
    "elfsight": {
      "type": "http",
      "url": "https://mcp.elfsight.com/mcp"
    }
  }
}
```

Windsurf (`mcp_config.json`):

```json
{
  "mcpServers": {
    "elfsight": {
      "serverUrl": "https://mcp.elfsight.com/mcp"
    }
  }
}
```

## What you can ask it

Reads:

- **`list_app_catalog`** — "Which Elfsight apps are there?"
- **`list_account_apps`** — "Which apps do I have in my workspace?"
- **`list_projects`** — "List my projects."
- **`list_widgets`** — "Show my Instagram Feed widgets in the `Landing` project."
- **`get_widget`** — "What are the current settings of widget `<id>`?"
- **`get_widget_limit`** — "How many more Google Reviews widgets can I create?"
- **`list_widget_templates`** — "What templates does the Pricing Table app have?"
- **`get_install_code`** — "Give me the embed code for this widget."
- **`get_widget_edit_links`** — "Give me the link to edit this widget in the Dashboard."

Writes:

- **`create_widget`** — "Create a Countdown Timer from the Black Friday template."
- **`duplicate_widget`** — "Make a copy of this widget."
- **`rename_widget`** — "Rename it to `Homepage reviews`."
- **`update_widget_settings`** — "Change the header text and the accent color."
- **`publish_widget`** — "Publish the changes."
- **`discard_widget_changes`** — "Throw away the unpublished changes."

### How widget changes work

- **Widgets are created from templates.** `create_widget` needs an app alias and an active
  template of that app (from `list_widget_templates`). The new widget is published right away;
  when the app's widget limit is reached it is saved as a draft (`status: DRAFT`).
- **Settings changes are drafts until published.** `update_widget_settings` applies a
  [JSON Merge Patch](https://www.rfc-editor.org/rfc/rfc7396) to the unpublished revision:
  objects merge, `null` removes a key, arrays are replaced as a whole. The site keeps showing
  the published version until `publish_widget` is called; `discard_widget_changes` drops the
  draft. The draft is shared with the Dashboard editor, so publishing or discarding also
  covers any unpublished edits you made there.
- **The publish link publishes on open.** `get_widget_edit_links` returns `editUrl` and
  `publishUrl`; opening `publishUrl` in a browser where you are signed in to Elfsight publishes
  the draft immediately.
- **Sensitive settings are protected.** Each app declares restricted settings (secrets,
  connected accounts and similar). Changes to them are rejected; edit those in the Dashboard.
- **Nothing is deleted.** The server has no tool that deletes widgets or projects.

## Privacy

Everything a tool returns can be sent to your AI provider (Anthropic, OpenAI, etc.) as part
of your conversation with the agent. Widget settings contain whatever you put in your widgets —
texts, links, images and, for some apps, contact details. Treat tool results accordingly.

## Limits and rate limiting

- `list_widgets` and `list_widget_templates` return pages of up to 100 items
  (`page`, `itemsPerPage`).
- Requests are rate-limited per client: 60 tool calls per minute. Going over the limit returns
  a `429` with a `Retry-After` header — wait and retry.

## Support

Questions or issues — contact [Elfsight Help Center](https://help.elfsight.com/) or your
account manager.
