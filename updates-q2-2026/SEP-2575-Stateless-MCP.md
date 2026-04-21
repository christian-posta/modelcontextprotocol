# Draft Q2 2026: Make MCP Stateless (SEP-2575)

## Overview

SEP-2575 represents a massive architectural shift for the Model Context Protocol, proposing that the protocol become **stateless-by-default**. 

Under the previous specification, MCP mandated a stateful initialization handshake (`initialize`) that established persistent session state between the client and the server for the duration of the connection. This SEP proposes removing that handshake entirely, moving MCP to a "pay-as-you-go" complexity model where each request is processed completely independently.

## The Core Problem

The mandatory stateful handshake created significant operational friction, especially in enterprise or cloud-native environments:

1. **Load Balancing Nightmare:** You could not place a simple stateless load balancer (L4/L7 round-robin) in front of an MCP server cluster. Because the client's capabilities and protocol version were negotiated once during the handshake and stored in memory on a specific backend server, load balancers were forced to use "sticky sessions" to route that client back to the exact same server instance.
2. **Poor Resilience:** If a backend server instance crashed or deployed a new version, all associated client sessions were lost. The client had to detect the failure, reconnect, and perform the entire initialization handshake again from scratch.

## Key Changes Proposed in SEP-2575

To solve these scalability and resilience issues, the SEP replaces the monolithic `initialize` handshake with discrete, stateless alternatives:

### 1. Per-Request Versioning & Capabilities
Instead of negotiating the protocol version and capabilities once at startup, the client must now pass this context with every single request.
* **Protocol Version:** Passed via the `MCP-Protocol-Version` HTTP header and the `io.modelcontextprotocol/protocolVersion` field in `_meta` (both MUST match). Version negotiation happens organically through `UnsupportedProtocolVersionError` responses.
* **Client Capabilities:** Passed via the `io.modelcontextprotocol/clientCapabilities` field in `_meta` on a per-request basis. Clients MUST also include `io.modelcontextprotocol/clientInfo` on every request.

### 2. The `server/discover` RPC (The "Look Before You Leap" Path)
A common question regarding per-request capabilities is: *"Doesn't this mean the client is flying blind and won't know what the server supports before calling a tool?"*

To solve this, SEP-2575 untangles the capability-exchange from the old `initialize` handshake and moves it into a dedicated `server/discover` RPC. Servers **MUST** implement this endpoint, but clients are not required to call it before making other requests.

* **Option A ("Look Before You Leap"):** If a complex client (like Claude Desktop) needs to know if a server supports advanced UI extensions, it calls `server/discover` first. The server returns its `supportedVersions`, `ServerCapabilities`, `serverInfo`, and optional `instructions`.
* **Option B ("Fly Blind"):** If a developer writes a simple automated script that just wants to execute a specific tool, it can skip discovery entirely and immediately fire the `tools/call` request. If the script uses a protocol version the server doesn't understand, the server won't crash—it will just return a standardized `UnsupportedProtocolVersionError` detailing the versions it *does* support.

### 3. Dedicated Streaming RPC (`subscriptions/listen`)
Under the old model, the Streamable HTTP transport required the client to open a persistent GET connection immediately upon connecting. The server would then use this single, open pipe to push everything: responses, logs, and server-to-client notifications (like `sampling/createMessage`). 

SEP-2575 replaces this monolithic pipe with a dedicated `subscriptions/listen` RPC for client-initiated streaming. The previous HTTP GET endpoint is **removed** — all communication uses POST.

#### Concrete Example: Streaming
If an AI client wants to subscribe to live resource updates (e.g., watching a log file), it no longer waits for the server to spontaneously push data down a global SSE connection. Instead, it explicitly calls `subscriptions/listen`, opting in to specific notification types.

**1. Client Explicitly Subscribes:**
```http
POST /mcp HTTP/1.1
MCP-Protocol-Version: 2025-06-18

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "subscriptions/listen",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2025-06-18",
      "io.modelcontextprotocol/clientInfo": { "name": "MyClient", "version": "1.0" },
      "io.modelcontextprotocol/clientCapabilities": {}
    },
    "notifications": {
      "resourcesListChanged": true,
      "resourceSubscriptions": ["file:///logs/system.log"],
      "logLevel": "info"
    }
  }
}
```

**2. Server Acknowledges, Then Streams Data:**
The server holds this HTTP request open as an SSE stream. The first event **MUST** be a `SubscriptionsAcknowledgedNotification` confirming which subscriptions the server accepted. Subsequent events carry the requested notifications.
```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"jsonrpc": "2.0", "method": "notifications/subscriptions/acknowledged", "params": {"notifications": {"resourcesListChanged": true, "resourceSubscriptions": ["file:///logs/system.log"], "logLevel": "info"}}}

data: {"jsonrpc": "2.0", "method": "notifications/resources/updated", "params": {"uri": "file:///logs/system.log"}}
```

This dedicated RPC ensures that streaming is strictly scoped to a specific client request with explicit opt-in per notification type, rather than bleeding into a global session state.

## Concrete Example

Here is how a standard tool call shifts from the old stateful model to the new stateless model.

### Old Stateful Model
The client connects and initializes. The server stores the client's capabilities in memory. Later, the client makes a request, relying on the server to remember the state.

**1. Handshake (Stored in Server Memory):**
```json
{
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": { "roots": {"listChanged": true} }
  }
}
```

**2. Tool Call (Relies on Memory):**
```json
{
  "method": "tools/call",
  "params": { "name": "get_weather" }
}
```

### New Stateless Model
The client skips initialization entirely. When it calls a tool, it bundles all necessary context into the request so *any* backend server behind the load balancer can safely process it.

**Tool Call (Self-Contained):**
```http
POST /mcp HTTP/1.1
MCP-Protocol-Version: 2025-06-18

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2025-06-18",
      "io.modelcontextprotocol/clientInfo": { "name": "MyClient", "version": "1.0" },
      "io.modelcontextprotocol/clientCapabilities": { "roots": {"listChanged": true} }
    }
  }
}
```

## Why this matters

By eliminating the `initialize` handshake, MCP traffic becomes inherently horizontally scalable. An enterprise can deploy 100 identical MCP servers behind a standard round-robin load balancer, and requests from a single AI client can be safely distributed across all of them without any state mismatch or sticky routing requirements.