# MCP Protocol and Specification Updates (Since Nov 2025 Release)

The upcoming release of the Model Context Protocol (MCP) introduces significant enhancements aimed at extending protocol capabilities, formalizing HTTP transport behaviors, improving client-server synchronization, and strengthening authorization flows. Below is a comprehensive summary of the updates going into the next release.

## 1. Core Protocol & Schema Additions

### Extensions Framework (SEP-2133)

The protocol now natively supports custom vendor configurations. The `ClientCapabilities` and `ServerCapabilities` schemas include a new `extensions` field (typed as `JSONObject`). This allows for safe, forward-compatible capability experimentation without modifying core schema strictness.

### MCP Apps - Interactive UIs (SEP-1865)

The specification introduces **MCP Apps**, enabling servers to deliver interactive frontend user interfaces directly within compatible MCP clients. This significantly broadens how users interact with server resources beyond text and basic prompts.

### HTTP Transport Standardization (SEP-2243)

Standardization of the Streamable HTTP transport has been formalized:

- **Required Headers:** `Mcp-Method` and `Mcp-Name` are now required on Streamable HTTP POST requests.
- **Custom Headers:** Introduced the `x-mcp-header` prefix, allowing servers to declare custom HTTP headers that clients should populate dynamically based on tool parameters.

### OpenTelemetry Trace Context (SEP-414)

Added standard conventions for distributed tracing across MCP boundaries. Servers and clients can now propagate OpenTelemetry context using `_meta` request parameters (`traceparent`, `tracestate`, `baggage`), enabling robust observability.

### Server Request Association (SEP-2260)

To prevent unsolicited or rogue server actions, all server-initiated requests (such as `roots/list` and `sampling/createMessage`) **MUST** now be explicitly associated with an active client request context.

### Tool Cache Optimization

Servers are now recommended (`SHOULD`) to return tools from `tools/list` in a **deterministic order**. This seemingly small change drastically improves client-side caching efficiency and maximizes LLM prompt cache hit rates.

### Schema Ergonomics

- **Task TTL:** Updated the generated JSON schema to explicitly allow `ttl: null` for Tasks to signify unlimited execution time, fixing strict-validation failures.
- **Sampling Definition (SEP-531):** Refined the sampling specification, strictly typing valid request and response fields for JSON-RPC messages.

## 2. Authorization & Security Enhancements

### Refresh Token Workflows (SEP-2207)

Added comprehensive guidance on implementing OIDC-flavored refresh token lifecycles, ensuring reliable long-lived connections for clients using robust identity providers.

### Step-up Authorization & Scope Accumulation (SEP-2350)

Resolved ambiguity in multi-step authorization scenarios. The spec clarifies how clients should accumulate minimal required scopes on the client side and defines the behavior for token challenges relying on hierarchical scopes.

### Multi-Authorization Server Migration (SEP-2352)

Added guidance for clients handling multi-AS behavior. It explicitly addresses how client credentials should be bound to specific authorization servers and provides safe practices for migration and token isolation.

### Client Application Types (SEP-837)

Strengthened `application_type` definitions during OIDC registration flows. The spec now dictates how locally-hosted web applications (accessed via `localhost`) must be classified as `native`, whereas `web` is strictly reserved for remote browser-based applications.

### Form-Mode Elicitation Scope

Clarified security constraints around "sensitive information" in Elicitation. The spec explicitly limits the strict `MUST NOT` directive to access credentials and tokens (passwords, API keys, payment data), leaving the collection of general PII up to the server's discretion.

## 3. Governance and Community Process Updates

While not strictly protocol technicalities, major structural improvements were merged to support ecosystem scaling:

- **PR-Based SEP Workflow (SEP-1850):** The Specification Enhancement Proposal process was completely migrated from GitHub issues to a traceable Pull Request model in a dedicated `seps/` directory.
- **Contributor Ladder (SEP-2148):** Established formal roles, advancement criteria, and decision delegation paths (from Contributor to Lead Core Maintainer).
- **Working Groups Charter (SEP-2149):** Defined a standardized template and governance rules for forming Working Groups and Interest Groups.
- **Succession & Amendment (SEP-2085):** Added formal procedures for protocol amendments and leadership succession.
- **SDK Tiering System:** The documentation site now ranks official and community SDKs in a Tiered assessment system (Tier 1-3) to convey readiness and capability support clearly.
# Draft/Pending SEPs (Upcoming Candidates)

The following SEPs are currently open Pull Requests and may be considered for the upcoming release or future releases. They are currently under discussion or review:

## Protocol & Architecture Overhauls

- **SEP-2575: Make MCP Stateless** - Proposes removing stateful connections in favor of a stateless protocol.
- **SEP-2567: Sessionless MCP via Explicit State Handles** - Proposes sessionless operations using explicit state handles.
- **SEP-2598: Pluggable Transports** - Enhancements to how transports are defined and plugged into the protocol.

## Features & Capabilities

- **SEP-2614: Add optional keywords field to Implementation for server routing** - Improves server routing and discovery by adding a keywords array to the Implementation metadata.
- **SEP: Resource Submission** - Client-to-server resource creation to improve agent coordination.
- **SEP-2564: Server-Side Filtering for List Methods** - Adds native filtering capabilities to `list` requests (e.g., `tools/list`, `resources/list`) to avoid transferring massive lists over the network.
- **SEP-2557: Adapt Tasks for Stateless & Sessionless Protocol** - Adjustments to the Tasks primitive to operate safely in sessionless/stateless environments.
- **SEP-2549: TTL for List Results** - Introduces Time-To-Live fields for list results to allow clients to better cache lists.
- **SEP-2532: Resource Streaming for Binary Content Delivery** - Native streaming primitives for large binary resources (like images or large files).
- **SEP: Event-Driven Tool Invocation** - Allows servers to push events that trigger LLM re-entry, turning tool invocation from purely client-driven to server-initiated.
- **SEP-2487: Add execution.requirements field to Tool** - Adds a field to explicitly define tool preconditions before a client attempts execution.
- **SEP-2433: Transfer Descriptors** - Support for Out-of-Band Data Transfer Negotiation.
- **SEP-2419: cache_hint well-known key** - Adds caching hints to `CallToolResult._meta` to improve LLM caching strategies.
- **SEP-2417: Model Preferences for Tools** - Allows tools to declare preferences for specific model types or characteristics.

## Telemetry, Auth & Lifecycle

- **SEP-2596: Specification Feature Lifecycle and Deprecation Policy** - Formalizes how features are added, matured, and deprecated within the spec.
- **SEP-2577: Deprecate Roots, Sampling, and Logging** - A proposal to deprecate several current protocol primitives.
- **SEP-2484: Require Conformance Tests for Standards Track SEPs** - Enforces conformance testing requirements for SEPs before they can reach "Final" status.
- **SEP-2468: Recommend Issuer (iss) Claim in MCP Auth Responses** - Updates auth guidance to recommend the OIDC `iss` claim.
- **SEP-2448: MCP server execution telemetry** - Extends telemetry capabilities for server execution.
