# Extension: ext-apps (MCP Apps)

**Repository:** [modelcontextprotocol/ext-apps](https://github.com/modelcontextprotocol/ext-apps)

## Overview

`ext-apps` provides a framework for interactive UI applications that render inside MCP hosts (like Claude, ChatGPT, VS Code, Postman). Instead of returning just text or static data, servers can return interactive HTML interfaces like data visualizations, complex forms, and rich dashboards directly in the chat interface.

## Key Advantages

1. **Context Preservation:** The app lives inside the conversation thread. Users don't need to switch tabs or lose their place.
2. **Bidirectional Data Flow:** The App can call any tool on the MCP server, and the host can push fresh results to the app.
3. **Host Integration:** The app can delegate actions to the host, routing requests through the user's existing connected capabilities (e.g., connected email providers or calendars) subject to user consent.
4. **Security & Sandboxing:** Apps run in a highly restricted, sandboxed `iframe` controlled by the host, ensuring they cannot access the parent page, steal cookies, or escape their container. Communication relies entirely on secure JSON-RPC over the `postMessage` API.

## How It Works

- A tool declares a UI resource via `_meta.ui.resourceUri` in its tool definition. This URI (using the `ui://` scheme) points to a standard MCP resource hosted by the server.
- When an LLM calls that tool, the host fetches the UI resource (HTML, JS, CSS) from the server. **Crucially, this is typically a static Single-Page Application (SPA) bundle.**
- The host renders the UI in a sandboxed `iframe`.
- Concurrently, the host executes the actual tool call on the server and receives the JSON result.
- **Dynamic Data Injection:** Instead of server-side rendering the HTML, the host pushes the JSON tool result directly into the running `iframe` via `postMessage`. The SPA then dynamically renders the data.
- The App and host continue to communicate using JSON-RPC 2.0 over `postMessage` for iframe-host communication. Lifecycle messages use a `ui/` prefix (e.g., `ui/initialize`, `ui/notifications/initialized`), while tool calls and other MCP operations reuse the existing MCP protocol methods.

## Host Orchestration & Graceful Degradation

Because the host client must explicitly orchestrate fetching the UI (`resources/read`) in parallel with executing the tool (`tools/call`), `ext-apps` requires explicit host support during initial capability negotiation. The host acts as the "glue" that intercepts the tool call, spins up the iframe, and pipes the resulting data into it.

If an MCP server with App-enabled tools connects to a client that _doesn't_ support `ext-apps` (like a basic CLI), it degrades gracefully. The client simply ignores the `_meta.ui` field, executes the standard tool call, and renders the raw text/JSON result as usual.

## Concrete Example: Tool with UI

**1. Server declares a tool with a UI resource:**
The tool definition includes `_meta.ui.resourceUri` pointing to an HTML application hosted by the server:

```json
{
  "name": "visualize_sales",
  "description": "Generates an interactive sales dashboard",
  "inputSchema": {
    "type": "object",
    "properties": {
      "quarter": { "type": "string" }
    }
  },
  "_meta": {
    "ui": {
      "resourceUri": "ui://dashboard/sales"
    }
  }
}
```

**2. Host orchestrates the tool call and UI fetch in parallel:**
When the LLM calls `visualize_sales`, the host does two things simultaneously:
- Executes `tools/call` to get the JSON result data from the server
- Fetches the SPA bundle via `resources/read` using the `ui://dashboard/sales` URI

**3. Host renders the iframe and injects data:**
The host renders the fetched HTML in a sandboxed `iframe`, then sends the `ui/initialize` message to the app. The app and host communicate via JSON-RPC 2.0 over `postMessage`:

Host → iframe (via `postMessage`): lifecycle initialization
```json
{
  "jsonrpc": "2.0",
  "method": "ui/initialize",
  "params": { }
}
```

iframe → Host: confirms ready
```json
{
  "jsonrpc": "2.0",
  "method": "ui/notifications/initialized"
}
```

Once initialized, the host pushes the tool result data into the running SPA. The app renders the interactive dashboard. From this point, the app can call tools on the server (routed through the host) and the host can push updated data into the app.

## Common Use Cases

- Exploring complex data (e.g., interactive maps, 3D globes, data heatmaps).
- Configuring complex setups with interdependent options (e.g., cloud deployments).
- Viewing rich media (e.g., PDF viewer, 3D models, sheet music).
- Multi-step workflows or real-time monitoring dashboards.

## Client & Framework Support

- Developers can use any web framework (React, Vue, Svelte, Vanilla JS, etc.).
- There is an `@modelcontextprotocol/ext-apps` App class wrapper, but it's not strictly required.
- Current client support includes ChatGPT, Claude, VS Code, Goose, Postman, and MCPJam.
