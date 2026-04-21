# Draft Q2 2026: Sessionless MCP via Explicit State Handles (SEP-2567)

## Overview

SEP-2567 is the complementary proposal to SEP-2575 (Stateless MCP). While SEP-2575 removes the protocol-level connection handshake, SEP-2567 proposes completely abolishing the concept of **MCP Sessions**. 

Under this proposal, implicit session-scoped state (like a shopping cart or a database connection tied to an `Mcp-Session-Id`) is entirely replaced by explicit, server-minted state handles that the AI model must carry and thread through subsequent tool calls.

## The Core Problem

After a year of production use, it became obvious that the concept of an "MCP Session" was fundamentally broken because no two clients treated it the same way:
* **ChatGPT** creates a completely fresh session for every single tool call.
* **Claude.ai** used to create a new session per tool call, but recently changed its behavior.
* **Desktop IDEs (Cursor/VS Code)** create one session when the app launches and keep it open all day.
* **Web interfaces** create one session per page load.

Because the lifetime of a session is completely unpredictable, an MCP Server author has no idea how long their data will survive. If a server ties a Playwright browser instance to the MCP session, it might stay open all day (IDE), or it might instantly be destroyed the millisecond the tool finishes returning data (ChatGPT). 

Furthermore, because tools *might* change based on session state, clients were forced to constantly re-fetch `tools/list` on every new session, completely ruining cross-session caching.

## Key Changes Proposed in SEP-2567

To fix this, SEP-2567 removes the session crutch and forces developers to manage state explicitly.

1. **Abolish Sessions:** The `Mcp-Session-Id` HTTP header is removed and all spec language describing session lifecycle and session-scoped behavior is deleted. The protocol is sessionless at every layer.
2. **Session-Independent List Endpoints:** With no session concept, the results of `tools/list`, `resources/list`, and `prompts/list` no longer have a per-session or per-connection scope to depend on. Lists can still change for other reasons (server deployment, auth changes); caching and invalidation mechanics are specified separately in SEP-2549 (server-advertised TTL + `notifications/*/list_changed`).
3. **Explicit State Handles (Guidance, Not Protocol):** The SEP recommends that servers manage cross-call state through explicit identifiers (e.g., `basket_id`) returned from creation tools and threaded through subsequent calls. This is a **tool-design pattern** the spec documents and recommends — there is no `handles/*` method, no handle type in the schema, and no wire-level concept of a handle.

## Concrete Example: Managing Application State

Imagine an MCP Server that manages an e-commerce shopping cart. Here is how it changes.

### Old Session-Scoped Model
The server relies on the hidden `Mcp-Session-Id` header to know whose cart is whose. If the client is ChatGPT (which closes the session immediately), the cart is instantly lost.

**Tool Call 1 (Create Cart):**
The AI calls `add_item(item="laptop")`. The server looks at the session ID, creates a cart in memory, and adds the laptop.

**Tool Call 2 (Checkout):**
The AI calls `checkout()`. The server looks at the session ID, finds the cart, and processes it. 

### New Explicit Handle Model
The server explicitly mints a handle (an ID) and forces the AI model to hold onto it. The AI decides how long that handle should live (e.g., across an entire conversation thread).

**Tool Call 1 (Create Cart):**
The AI calls the `create_basket()` tool. The server creates a database record and returns an opaque identifier in both a human-readable `content` field and a machine-readable `structuredContent` field.
* **Server returns:**
```jsonc
{ "content": [{ "type": "text", "text": "Created basket bsk_a1b2c3" }],
  "structuredContent": { "basket_id": "bsk_a1b2c3" } }
```

**Tool Call 2 (Add Item):**
The AI calls `add_item()`, but now it must explicitly thread the handle back to the server.
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "add_item",
    "arguments": {
      "basket_id": "bsk_a1b2c3",
      "sku": "shoes"
    }
  }
}
```

## Why this matters

By removing sessions and recommending explicit handles, the protocol achieves three key benefits:
1. **Cacheability:** `tools/list` results no longer vary per session, so clients can cache them at the deployment/auth level and invalidate via TTL or `notifications/*/list_changed` (per SEP-2549). This eliminates the `O(subagents x servers)` re-fetch cost the SEP identifies.
2. **Predictability:** Server authors no longer have to guess what a "session" means. Handle lifetime is determined by the server's documented durability policy (e.g., "baskets expire after 24h idle") and the model's ability to thread the handle through calls.
3. **Agent Orchestration:** It solves strict cardinality constraints. With sessions, one connection meant exactly one scope. Now, a master AI orchestrator can spin up three different sub-agents, pass them all the same `basket_id` so they can share a cart, but give them different `browser_id`s so their web-scraping state is isolated.