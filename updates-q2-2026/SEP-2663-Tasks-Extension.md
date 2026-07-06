# Draft Q2 2026: Tasks moved to an Official Extension (SEP-2663)

## Overview

Tasks — MCP's "call now, fetch the result later" async execution mode — shipped **experimentally in the core protocol** in the `2025-11-25` release (via SEP-1686). SEP-2663 **pulls Tasks out of the core schema entirely** and republishes it as the first official, versioned MCP **extension**: `io.modelcontextprotocol/tasks`.

This is two changes in one:

1. A **packaging change** — Tasks is no longer part of `schema/draft/schema.ts`; it lives in the extensions mechanism formalized by SEP-2133 (the `extensions` field on capabilities). The intent is to let Tasks keep iterating off the core release cadence and be promoted back into core once it stabilizes.
2. A **redesign** — the Tasks API was reshaped to survive in the new sessionless/stateless world (SEP-2567/2575), to comply with SEP-2260 (no unsolicited server→client requests), and to plug into MRTR (SEP-2322).

Status: **Final**, Extensions Track. Extension ID `io.modelcontextprotocol/tasks`, targeting the next spec release. Authors: Chang, McCaffrey, Agents WG.

## The Core Problem

The original core Tasks design (SEP-1686) baked several assumptions that broke once sessions and server-initiated requests were removed:

- **`tasks/list` needed a scope.** Listing "your tasks" implicitly meant "the tasks for this session." With sessions gone (SEP-2567), there is no safe, well-defined set to enumerate — a `taskId` becomes an unguessable bearer handle instead.
- **Client-generated task IDs and per-request opt-in** meant the client had to know up front (and re-fetch `tools/list`) which tools supported tasks.
- **The blocking `tasks/result` call and client-hosted tasks** relied on the same server→client callback machinery that SEP-2260 outlaws.

## Key Changes Introduced by SEP-2663

### 1. Tasks is now negotiated as an extension, not a core capability

Previously a `tasks` capability lived in `ClientCapabilities`/`ServerCapabilities`. Now the server advertises support by listing the extension in its `extensions` capability (returned from `server/discover`):

```json
{
  "extensions": {
    "io.modelcontextprotocol/tasks": {}
  }
}
```

The client likewise declares the extension in its per-request `_meta` capabilities. There is no per-request `task` flag and no `tools/list` warmup.

### 2. The server decides when to make a task (was: the client)

In the old model the **client** asked for a task (setting a `task` param / `_meta` and generating the `taskId`). In the new model the **client only declares that it can handle tasks**; the **server** decides per request whether to run synchronously or return a task handle, and the **server mints the `taskId`**.

The signal is the new MRTR-style discriminator: a task response is a `CreateTaskResult` with `resultType: "task"`, returned in place of (e.g.) a `CallToolResult`.

### 3. Method surface changed

| Old (SEP-1686 core) | New (SEP-2663 extension) | Notes |
|---------------------|--------------------------|-------|
| `tasks/get`         | `tasks/get`              | Now **inlines the result** when the task is `completed` (no separate fetch). |
| `tasks/result` (blocking) | **removed**        | Replaced by polling `tasks/get`; input-required flows use `tasks/input_response`. |
| `tasks/cancel` (via `notifications/cancelled`) | `tasks/cancel` (dedicated method) | `notifications/cancelled` **MUST NOT** be used for tasks. |
| `tasks/list`        | **removed**              | No session scope to list against. |
| `tasks/delete`      | **removed**              | Cleanup is TTL-based only. |
| —                   | `tasks/update`           | **new** — client→server input during execution. |
| `notifications/tasks/status` | `notifications/tasks` | push updates, opt-in via `subscriptions/listen`. |

### 4. Field and state cleanup

- States reduced to **five**: `working`, `input_required`, `completed`, `cancelled`, `failed` (the old `submitted` and `unknown` are gone).
- Field renames: `keepAlive` → **`ttlMs`** (number or `null` = unlimited), `pollFrequency` → **`pollIntervalMs`**.
- New fields: `createdAt`, `lastUpdatedAt` (ISO 8601), `statusMessage`.
- `failed` is reserved for JSON-RPC/protocol errors only; a *tool* error (`isError: true`) is still a `completed` task with the error result inlined.

### 5. `taskId` is now a security-bearing handle

With no session to scope it, the `taskId` is an unguessable capability token. The server MUST generate it with sufficient entropy and MUST run auth/authz checks on **every** `tasks/*` request. Over Streamable HTTP the client MUST set the `Mcp-Name` header to the `taskId` (per SEP-2243) so intermediaries can route the request to the instance holding the task's state — this replaces session affinity.

## Concrete Example: a long-running tool call

### Old Core-Tasks Model (SEP-1686)

Client opts in and supplies its own id:

```jsonc
{ "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": { "name": "deep_research", "arguments": { "topic": "..." },
    "_meta": { "modelcontextprotocol.io/task": { "taskId": "client-generated-123" } } } }
```

Later it **blocks** on `tasks/result` to get the answer.

### New Extension Model (SEP-2663)

**1. Client calls the tool normally** (having declared the `io.modelcontextprotocol/tasks` extension). The server chooses to make it a task:

```jsonc
{
  "jsonrpc": "2.0", "id": 1,
  "result": {
    "resultType": "task",
    "taskId": "task_9f3c",
    "status": "working",
    "createdAt": "2026-07-05T12:00:00Z",
    "lastUpdatedAt": "2026-07-05T12:00:00Z",
    "ttlMs": 3600000,
    "pollIntervalMs": 5000
  }
}
```

**2. Client polls `tasks/get`** (setting `Mcp-Name: task_9f3c`), honoring `pollIntervalMs`, until the status is terminal. When `completed`, `tasks/get` returns the result inline:

```jsonc
{
  "jsonrpc": "2.0", "id": 2,
  "result": {
    "resultType": "complete",
    "taskId": "task_9f3c",
    "status": "completed",
    "result": { "content": [ { "type": "text", "text": "…research report…" } ] }
  }
}
```

If the task needs input mid-flight, it enters `input_required`; the client fetches the `inputRequests` and answers via `tasks/input_response` (the persistent half of MRTR — see SEP-2322).

## Why this matters

Tasks is the escape hatch for everything that *can't* be stateless — genuinely long operations (deep research, CI runs, batch jobs, agent-to-agent handoffs). Moving it to an extension keeps the core protocol lean and lets the async model mature without dragging the whole spec's version along with it. The server-directed redesign is also more honest: the server, not the client, actually knows whether an operation will be slow, so it should be the one to decide to hand back a handle.

## Challenges for Existing Servers

1. **Not wire-compatible with the old design.** Legacy `tasks/result` now returns `-32601` (method not found); the old `task` param is ignored; `tasks.*` capabilities must migrate to the `io.modelcontextprotocol/tasks` extension namespace.
2. **Durable creation before responding.** The server MUST NOT return `CreateTaskResult` until the task is durably created — i.e. a subsequent `tasks/get` from another instance would resolve it. On eventually-consistent stores this means waiting for consistency before replying.
3. **Routing without sessions.** Because state is server-side but sessions are gone, the `Mcp-Name = taskId` routing convention is load-bearing. Deployments must actually honor it in their gateways/load balancers.
4. **Cooperative, eventually-consistent cancel/update.** `tasks/cancel` and `tasks/update` are ack-only; the server may still finish work after a cancel. Clients must treat cancellation as a request, not a guarantee, and keep polling to learn the real terminal state.
