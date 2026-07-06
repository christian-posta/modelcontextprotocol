# Draft Q2 2026: Deprecate Roots, Sampling, and Logging (SEP-2577)

## Overview

SEP-2577 formally deprecates three long-standing client/server features: **Roots**, **Sampling**, and **Logging**. Nothing is removed in this draft — every type and method stays fully functional — but all three are marked `@deprecated`, new implementations are told not to adopt them, and they are scheduled for removal under the new feature-lifecycle policy (SEP-2596).

This is the "prune the surface area" companion to the stateless/MRTR work. Two of the three deprecated features (Roots and Sampling) were exactly the server→client callbacks that MRTR (SEP-2322) had to contort itself to preserve; deprecating them signals the long-term direction is to stop relying on the client as a callable service at all.

Status: **Final**, Standards Track. Author: Kurtis Van Gent. Predicated on the one-year-per-version support policy from SEP-2596.

## The Core Problem

All three features share the same profile: **high complexity, low adoption, and a mature out-of-band alternative.**

- **Roots** (`roots/list`, `notifications/roots/list_changed`): semantics were always vague — "here are some filesystem roots the user cares about" — and mostly informational. Passing a directory or file explicitly as a tool argument or resource URI is clearer and doesn't require the server to interrogate the client.
- **Sampling** (`sampling/createMessage`): the most complex and most security-sensitive feature in MCP. It asks the client to run an LLM call on the server's behalf, which drags in human-in-the-loop approval, model selection/preferences, and (post SEP-1577) tool-use loops. Adoption stayed low, and most servers that need an LLM just call a provider API directly.
- **Logging** (`logging/setLevel`, `notifications/message`): a bespoke log-shipping channel that never matched the maturity of `stderr` (for stdio) or OpenTelemetry (for everything else).

## Key Changes Introduced by SEP-2577

### 1. What gets the `@deprecated` marker

| Feature | Methods / notifications | Capabilities | Representative types |
|---------|-------------------------|--------------|----------------------|
| **Roots**   | `roots/list`, `notifications/roots/list_changed` | `ClientCapabilities.roots` | `Root`, `ListRootsRequest`, `ListRootsResult` |
| **Sampling**| `sampling/createMessage` | `ClientCapabilities.sampling` (and `tasks.requests.sampling`) | `CreateMessageRequest/Result`, `SamplingMessage`, `ModelPreferences`, `ModelHint`, `ToolChoice`, `ToolUseContent`, `ToolResultContent` |
| **Logging** | `logging/setLevel`, `notifications/message` | `ServerCapabilities.logging` | `LoggingLevel`, `SetLevelRequest`, `LoggingMessageNotification` |

> Note the overlap with the stateless work: independently of this deprecation, SEP-2575 **removes** `logging/setLevel` (log level is now set per-request via `io.modelcontextprotocol/logLevel` in `_meta`) and removes `notifications/roots/list_changed`. So Logging and Roots are simultaneously being *slimmed* by 2575 and *deprecated wholesale* by 2577.

### 2. It is explicitly **not** a breaking change (yet)

- No types are removed. Everything keeps working during the deprecation window.
- The union types that reference these (`ClientNotification`, `ClientResult`, `ServerRequest`, `ServerNotification`) **MUST NOT** be modified yet — removing the members now would be the breaking change.
- Versions released within one year MUST continue to include these features (as deprecated). Removal only becomes permissible in a spec version released more than a year later.

### 3. Suggested migrations

- **Roots →** pass directories/files via tool parameters, resource URIs, or server configuration.
- **Sampling →** integrate directly with an LLM provider API server-side.
- **Logging →** write to `stderr` (stdio) or emit OpenTelemetry (aligns with SEP-414 trace context).

## Concrete Example: what "deprecated" looks like

### Before (2025-11-25)

A client advertises full support and the server freely calls back for sampling:

```jsonc
// Client capabilities (negotiated at initialize)
{ "capabilities": { "roots": { "listChanged": true }, "sampling": {} } }

// Server -> Client callback, mid tool-call
{ "jsonrpc": "2.0", "id": "s1", "method": "sampling/createMessage",
  "params": { "messages": [ /* ... */ ], "maxTokens": 100 } }
```

### After (this draft)

The types still exist and still validate — but they carry `@deprecated`, SDKs SHOULD warn when the capability is negotiated, and the *mechanism* has already shifted: a server that still wants sampling does it through MRTR's `input_required` result (SEP-2322), not a reverse-direction request. New servers are told to skip all of this and call an LLM API directly.

```jsonc
// Schema (illustrative): the type is retained but annotated
/** @deprecated Deprecated in 2026-07-28 per SEP-2577. Prefer direct LLM provider APIs. */
interface CreateMessageRequest { /* unchanged shape */ }
```

## Why this matters

Deprecation policy is how a protocol stays honest about its scope. These three features carried a disproportionate share of MCP's complexity and security surface (Sampling is the prime prompt-injection/exfiltration vector; Roots leaks filesystem structure) for relatively little real-world use. Marking them deprecated — with a concrete 12-month clock (SEP-2596) — lets the ecosystem stop building new dependencies on them now, while giving existing implementations a guaranteed runway. Net-positive for security, and it clears the path for the stateless/MRTR architecture to eventually drop the whole notion of the server calling the client.

## Challenges for Existing Servers

1. **Don't rip it out yet.** The correct behavior *this* release is to keep supporting these features (they're deprecated, not removed) and to emit a warning when a deprecated capability is negotiated — not to reject it.
2. **Watch the double-move on Logging/Roots.** Because 2575 already changed *how* logging and roots-change notifications work, an implementation updating to this draft needs to both (a) move to per-request `io.modelcontextprotocol/logLevel` and drop `roots/list_changed`, and (b) treat the remaining roots/sampling/logging surface as deprecated. Read 2575 and 2577 together.
3. **Plan the migration now.** Any product currently depending on Sampling as its LLM path should start moving to a direct provider integration; anything using Roots for directory discovery should move that into explicit tool arguments before the removal window opens.
