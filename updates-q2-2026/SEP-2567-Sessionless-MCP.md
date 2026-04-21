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

1. **Abolish Session Primitives:** `session/create`, `session/destroy`, and the `Mcp-Session-Id` HTTP header are completely removed from the protocol.
2. **Cacheable List Endpoints:** Endpoints like `tools/list`, `resources/list`, and `prompts/list` **MUST NOT** depend on per-connection or prior-tool-call state. Because they are now fully static, clients can cache the tool list globally at the deployment/auth level.
3. **Explicit State Handles:** Application state must now be managed through explicit identifiers passed back and forth between the client and server.

## Concrete Example: Managing Application State

Imagine an MCP Server that manages an e-commerce shopping cart. Here is how it changes.

### ❌ Old Session-Scoped Model
The server relies on the hidden `Mcp-Session-Id` header to know whose cart is whose. If the client is ChatGPT (which closes the session immediately), the cart is instantly lost.

**Tool Call 1 (Create Cart):**
The AI calls `add_item(item="laptop")`. The server looks at the session ID, creates a cart in memory, and adds the laptop.

**Tool Call 2 (Checkout):**
The AI calls `checkout()`. The server looks at the session ID, finds the cart, and processes it. 

### ✅ New Explicit Handle Model
The server explicitly mints a handle (an ID) and forces the AI model to hold onto it. The AI decides how long that handle should live (e.g., across an entire conversation thread).

**Tool Call 1 (Create Cart):**
The AI calls the `create_basket()` tool. The server creates a database record and returns a string identifier.
* **Server returns:** `"basket_id_99283"`

**Tool Call 2 (Add Item):**
The AI calls `add_item()`, but now it must explicitly thread the handle back to the server.
```json
{
  "method": "tools/call",
  "params": {
    "name": "add_item",
    "arguments": {
      "basket_id": "basket_id_99283",
      "item": "laptop"
    }
  }
}
```

## Why this matters

By forcing state into explicit handles, the protocol achieves three massive benefits:
1. **Cacheability:** The list of tools never changes based on user actions, so `tools/list` can be fetched once and cached indefinitely by the client.
2. **Predictability:** Server authors no longer have to guess what a "session" means. State lives exactly as long as the AI model remembers the handle and chooses to pass it.
3. **Agent Orchestration:** It solves strict cardinality constraints. Previously, one connection meant exactly one session. Now, a master AI orchestrator can spin up three different sub-agents, pass them all the same `basket_id` so they can share a cart, but give them different `browser_id`s so their web-scraping state is isolated.