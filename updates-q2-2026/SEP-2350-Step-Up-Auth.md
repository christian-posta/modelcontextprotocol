# Updates Q2 2026: Step-Up Authorization & Scope Accumulation (SEP-2350)

## Overview

SEP-2350 clarifies how MCP handles "Step-Up Authorization"—the process where a client already has a valid token, but encounters a 403 Forbidden error because it needs *additional* permissions (scopes) to complete a new action. 

Historically, the specification was ambiguous about whose job it was to remember the client's previous permissions when asking for new ones. This led to a confusing dynamic where clients would "lose" their old permissions after stepping up to get new ones. SEP-2350 firmly resolves this by establishing **client-side scope accumulation** and allowing servers to remain completely stateless regarding client scope sets.

## The Core Problem

Imagine an AI client starts a session and requests `files:read` access. Later, the user asks the AI to edit a file. The client calls the `edit_file` tool, but the server rejects it because it needs `files:write`.

Before this SEP, the server guidance suggested that the server itself should try to remember what the client already had and append the new requirement:
* **Old Server Response:** `WWW-Authenticate: Bearer error="insufficient_scope", scope="files:read files:write"`

This was incredibly difficult for servers to implement because it forced the Resource Server to maintain state about exactly what scopes every single client token possessed, just so it could echo them back in the error message. If the server forgot to include `files:read` in the error response, the client would ask the Auth Server *only* for `files:write`. The Auth Server would issue a new token with *only* write access, and the client would mysteriously lose its ability to read files!

## Key Changes Introduced by SEP-2350

SEP-2350 aligns MCP with standard OAuth behavior (RFC 6750) by shifting the accumulation burden entirely to the client.

### 1. Stateless Server Challenges
When an MCP Server rejects a request due to insufficient scope, the server reports the scopes needed for the current operation — not the union of previously granted scopes. The spec offers a range of approaches: a **minimum approach** (only the scopes for the triggering operation), a **recommended approach** (the operation's scopes plus related scopes that commonly work together), and an **extended approach** (additionally including scopes the server anticipates the client may need soon). In all cases, the server no longer needs to track the client's existing scope set.
* **New Server Response:** `WWW-Authenticate: Bearer error="insufficient_scope", scope="files:write"`

### 2. Client-Side Scope Accumulation
Scope accumulation across operations is now explicitly a **client-side responsibility**.
* When a client receives a 403 `insufficient_scope` challenge, it **SHOULD** compute the union of its *previously requested scopes* and the *newly challenged scopes*.
* The client then initiates the re-authorization flow asking the Auth Server for the combined set (`files:read files:write`).

### 3. Reducing Step-Up Round Trips
Regardless of which approach (minimum, recommended, or extended) a server chooses, it **SHOULD** include all scopes required for the current operation in a single challenge. Challenging incrementally — returning one missing scope, then another on the subsequent retry — forces multiple authorization round-trips for a single operation and degrades user experience.

The spec's **recommended approach** goes further: servers include the operation's scopes along with related scopes that commonly work together, reducing round trips across *related* operations. For example, if a `save_document` tool requires both `files:write` and `user:profile`, the server should return `scope="files:write user:profile"` in a single challenge rather than requiring separate step-up flows for each scope.

## Concrete Example

Here is how the interaction works under the new SEP-2350 rules:

**1. The Client makes a request with limited scopes:**
The client currently holds a token with `files:read`. It attempts to execute a tool that requires writing.
```http
POST /mcp HTTP/1.1
Authorization: Bearer <token_with_read_only>
Content-Type: application/json

{"jsonrpc": "2.0", "id": 1, "method": "tools/call", "params": {"name": "save_document"}}
```

**2. The Server issues a stateless challenge:**
The server rejects the request. It does *not* care what scopes the client currently has. It simply states the requirements for the `save_document` tool.
```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope",
                         scope="files:write",
                         resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource",
                         error_description="File write permission required for this operation"
```

**3. The Client computes the union and steps up:**
The client intercepts the 403. It looks at its own internal state: *"I previously asked for `files:read`. The server now wants `files:write`."*
The client computes the union and opens the user's browser to request the combined set:
```http
GET /authorize?
  client_id=my-ai-client&
  scope=files:read files:write&
  ...
```

## Multi-Step Accumulation Example

In a real session, a client may step up multiple times. Here is how its internal scope state evolves:

```
Step 1: Initial authorization
  Client requests: scope="files:read"
  Token granted:   files:read
  Client state:    ["files:read"]

Step 2: User asks AI to edit a file → 403 insufficient_scope, scope="files:write"
  Client computes union: ["files:read"] ∪ ["files:write"] = ["files:read", "files:write"]
  Client requests: scope="files:read files:write"
  Token granted:   files:read files:write
  Client state:    ["files:read", "files:write"]

Step 3: User asks AI to generate a report → 403 insufficient_scope, scope="reports:create"
  Client computes union: ["files:read", "files:write"] ∪ ["reports:create"]
  Client requests: scope="files:read files:write reports:create"
  Token granted:   files:read files:write reports:create
  Client state:    ["files:read", "files:write", "reports:create"]
```

At each step, the server only tells the client what *that specific operation* needs. The client is responsible for remembering everything it has previously requested and computing the union. Without this, Step 3 would result in a token with *only* `reports:create`, silently losing the file permissions.

## Note on Hierarchical Scopes

Some enterprise Authorization Servers define scope hierarchies (e.g., an `admin` scope automatically implies `read` and `write`). 

SEP-2350 explicitly notes that clients **do not need to deduplicate** hierarchically. If a client currently has an `admin` scope, and a tool throws a 403 asking for a `read` scope, the client simply computes the raw string union (`scope=admin read`) and sends it to the Auth Server. Authorization servers typically normalize such redundancy during token issuance. This keeps client implementation simple and logic-free.