# Updates Q2 2026: Extension Framework (SEP-2133)

https://modelcontextprotocol.io/seps/2133-extensions

## Overview

SEP-2133 officially introduces an Extension Framework for the Model Context Protocol (MCP). It establishes a governance model, lifecycle, and presentation structure for extensions, allowing the MCP ecosystem to scale and experiment with new capabilities without forcing changes into the core protocol.

## Key Changes Introduced by SEP-2133

### 1. Capability Negotiation

The most significant technical addition is the `extensions` field in both `ClientCapabilities` and `ServerCapabilities` schemas (represented as a `JSONObject`).

- **Format:** Extensions use a unique identifier format: `{vendor-prefix}/{extension-name}` (e.g., `io.modelcontextprotocol/oauth-client-credentials` or `com.example/websocket-transport`).
- **Versioning & Breaking Changes:** Breaking changes MUST use a new identifier, e.g. `io.modelcontextprotocol/oauth-client-credentials-v2`.
- **Settings Objects:** Each key within the `extensions` map maps to a custom configuration/settings object defined entirely by the extension. An empty object indicates no settings.

**Example: Client Advertising Extension Support**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "roots": {
        "listChanged": true
      },
      "extensions": {
        "io.modelcontextprotocol/ui": {
          "mimeTypes": ["text/html;profile=mcp-app"]
        }
      }
    },
    "clientInfo": {
      "name": "ExampleClient",
      "version": "1.0.0"
    }
  }
}
```

**Example: Server Acknowledging Extension Support**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": {},
      "extensions": {
        "io.modelcontextprotocol/ui": {}
      }
    },
    "serverInfo": {
      "name": "ExampleServer",
      "version": "1.0.0"
    }
  }
}
```

**Example: Server-Side Capability Checking**
Servers SHOULD check client capabilities before offering extension-specific features:

```typescript
const hasUISupport = clientCapabilities?.extensions?.[
  "io.modelcontextprotocol/ui"
]?.mimeTypes?.includes("text/html;profile=mcp-app");

if (hasUISupport) {
  // Register tools with UI features
} else {
  // Register text-only fallback
}
```

### 2. Extension Tiers & Governance

SEP-2133 sets up a formal incubation pipeline for features:

- **Official Extensions:** Maintained within the MCP GitHub organization (using prefixes like `ext-`). Identified by the `io.modelcontextprotocol/` vendor prefix. They are fully endorsed and recommended by core maintainers. Examples cited in the SEP include the `ext-auth` and `ext-apps` repositories.
- **Experimental Extensions:** Incubated inside repositories starting with `experimental-ext-` (e.g., `experimental-ext-interceptors`). They are managed by Working Groups (WGs) or Interest Groups (IGs) and act as a prototype sandbox before they undergo a formal SEP process (Extensions Track) to become Official.
- **Unofficial Extensions:** Built and governed entirely by external developers, utilizing their own reversed domain name prefix (e.g., `com.example/websocket-transport`).

### 3. Graceful Degradation & Opt-In Architecture

- **Opt-in by Default:** Where extensions are supported by SDKs or servers, they **MUST** be disabled by default, requiring explicit developer/user opt-in.
- **Fallback Behaviors:** If an extension is requested but not mutually supported, implementations **MUST** either degrade gracefully to core protocol behavior or reject the request (if the extension is strictly mandatory). For example, a server offering UI-enhanced tools should still return meaningful text content for clients that do not support the UI extension.

### 4. SDK Support & Legal Protections

- SDK maintainers are under _no obligation_ to implement extensions to maintain 100% core protocol conformance.
- Official extensions are required to be licensed under Apache 2.0 with a clear Contributor License Grant, ensuring safe antitrust and IP protection.

## Why this matters

By modularizing MCP, developers can start standardizing advanced functionality like Enterprise Identity Authentication (`ext-auth`) or interactive UI rendering (`ext-apps`) without bloating the core specification. It acts as a pressure release valve for complex feature requests that are only relevant to specific sectors of the community.

## Roadmap & Future Extensions

Following the successful release of `ext-auth` and `ext-apps`, several related ecosystem efforts are underway:

1. **Skills Over MCP Interest Group:** A "Skills Over MCP" Interest Group has been chartered (PR #2568, merged) to explore composing multiple capabilities into reusable bundles. A prior SEP proposing "Agent Skills as a First-Class MCP Primitive" (PR #2076) was closed, but community interest continues through this IG.
2. **Registry Working Group:** A Registry WG charter is currently in progress (PR #2587, open), which would provide infrastructure for discovering servers and extensions based on their capabilities.
3. **Enterprise Auth Extensions:** Enterprise-focused work is already landing via `ext-auth` (enterprise-managed authorization, OAuth client credentials). Additional enterprise governance (audit logging, policy controls) may emerge as further extensions, though no dedicated Enterprise WG has been formally chartered.
