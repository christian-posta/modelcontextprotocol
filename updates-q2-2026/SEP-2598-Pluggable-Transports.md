# Draft Q2 2026: Pluggable Transports (SEP-2598)

## Overview

SEP-2598 formalizes an official, SDK-layer extension point for creating and distributing **Custom Transports**. 

By default, the Model Context Protocol defines two Standard transports: `stdio` and Streamable HTTP. However, there has been immense pressure from the community to add more (e.g., WebSockets for serverless, gRPC for high-throughput enterprise, SSH for remote execution). This SEP provides a path to build and share these transports without bloating the core specification.

## The Core Problem

Currently, if someone wants a new transport (like WebSockets), they draft a SEP asking for it to be included in the core specification. This creates an unsustainable burden:

1. **Specification Complexity:** Every transport admitted into the core adds another chapter of rules (framing, security, lifecycle edge cases) that the specification maintainers must own and keep consistent forever.
2. **SDK Maintainer Burden (The N-Multiplier):** If gRPC is added to the core spec, it becomes a mandatory implementation requirement for *every* official SDK (Python, TypeScript, Go, Java, etc.). A single spec addition forces the core SDK maintainers to do N parallel implementations and maintain them indefinitely.

## Key Changes Proposed in SEP-2598

To stop the core specification from exploding in size, SEP-2598 formalizes a "Pluggable Transports" architecture at the SDK layer.

### 1. The SDK `Transport` Interface
The core specification is relieved of having to define new transports. Instead, every Tier 1 MCP SDK **MUST** expose a public, stable `Transport` interface (along with a conformance test harness).

### 2. Third-Party Distribution
Concrete transport implementations (WebSocket, gRPC, SSH) are distributed as third-party packages independent of the core SDK. If a transport becomes incredibly popular, it can be formalized as an Official Extension under SEP-2133, but it remains external to the core SDK conformance tiering.

### 3. Wire Format Flexibility
Historically, custom transports were forced to use the exact JSON-RPC wire format. SEP-2598 relaxes this rule. Custom transports **MAY** use any on-the-wire encoding they want (e.g., binary Protobufs for gRPC), as long as they strictly preserve MCP semantics. 

In exchange for this flexibility, the custom transport **MUST** publish a bidirectional mapping back to JSON-RPC so that generic translating proxies can bridge the custom transport back to standard MCP traffic if necessary.

## Concrete Example: Connecting via a Pluggable Transport

Here is how a developer's experience changes under the Pluggable Transports model.

Instead of waiting for the core team to merge WebSocket support into the official Python SDK, the community builds a compliant transport package. The developer installs both the core SDK and the plugin, and simply injects the plugin into the client at construction time — the session layer is unaware of which transport is in use.

> **Note:** The SEP's reference implementation is still in progress (the PR marks it TODO). The exact package names, import paths, and API shapes have not yet been defined. The pattern below is illustrative of the intended developer experience, not a specification of the final API.

```python
# Illustrative pseudocode — actual API shape TBD in the reference implementation

# 1. Import the Core SDK
from mcp.client.session import ClientSession
# 2. Import a community-built transport plugin
from <community_websocket_package> import WebSocketTransport

# 3. Construct the transport and inject it into the client
transport = WebSocketTransport("wss://api.example.com/mcp")
async with ClientSession(transport) as session:
    # The session API is identical regardless of which transport is used
    result = await session.call_tool("get_weather", {"city": "Seattle"})
```

## Why this matters

This solves the bottleneck of protocol evolution. It allows the core MCP spec to remain lean, secure, and focused entirely on AI context bridging. Meanwhile, enterprise infrastructure teams can build wildly optimized, custom transports (like native gRPC streams) and plug them directly into the standard MCP SDKs without waiting for permission or burdening the core maintainers.