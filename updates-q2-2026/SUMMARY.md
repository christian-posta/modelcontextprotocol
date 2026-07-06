# MCP Protocol & Specification Updates — Q2 2026 Draft/RC

This summarizes the changes going into the next MCP release, as reflected in the officially
published **draft** at <https://modelcontextprotocol.io/specification/draft> (protocol version
**`2026-07-28`**). The authoritative "Key Changes" list is the spec's own
[`changelog.mdx`](https://modelcontextprotocol.io/specification/draft/changelog).

All SEPs: <https://modelcontextprotocol.io/seps> · Release plan: <https://plan.modelcontextprotocol.io>

> **Reading note.** This draft is a *major architectural overhaul*, not an incremental feature drop.
> Three mega-SEPs — **Stateless (2575)**, **MRTR (2322)**, and **Tasks-as-Extension (2663)** — reshape
> the rest. The overarching theme is *make every request self-contained and independently routable, and
> prune the surface area that prevented that* (sessions, the init handshake, server→client callbacks,
> SSE resumability, roots/sampling/logging).

Each item below links to a dedicated note file with before→after detail.

---

## 🔴 Biggest changes (architectural / breaking — read these first)

### 1. Make MCP Stateless (SEP-2575) — `SEP-2575-Stateless-MCP.md`
Removes the `initialize` / `notifications/initialized` handshake. Every request now carries its own
`io.modelcontextprotocol/protocolVersion`, `clientInfo`, and `clientCapabilities` in `_meta`. Adds the
mandatory **`server/discover`** RPC for up-front capability/version discovery, replaces the HTTP GET
endpoint + `resources/subscribe`/`unsubscribe` with a single **`subscriptions/listen`** stream, removes
`ping`, `logging/setLevel`, and `notifications/roots/list_changed`, and drops SSE resumability
(`Last-Event-ID`). Version mismatch → `UnsupportedProtocolVersionError`.

### 2. Sessionless MCP via Explicit State Handles (SEP-2567) — `SEP-2567-Sessionless-MCP.md`
Removes the `Mcp-Session-Id` header and all session-lifecycle language. List endpoints no longer vary
per connection (making them cacheable — see TTL below). Cross-call state moves to explicit, server-minted
handles threaded through tool arguments (a documented pattern, not a wire primitive).

### 3. Multi Round-Trip Requests / MRTR (SEP-2322) — `SEP-2322-MRTR.md` *(was missing from prior review)*
Replaces all **server-initiated requests** (sampling / elicitation / roots callbacks). Instead of calling
back to the client, a server returns an `InputRequiredResult` (`resultType: "input_required"`) listing
what it needs; the client fulfills it and **retries the original request** with `inputResponses` + an
opaque `requestState`. Also adds a **required `resultType` discriminator on every result**
(`"complete"` | `"input_required"` | `"task"`). This is what makes stateless/sessionless actually work.

### 4. Tasks moved to an Official Extension (SEP-2663) — `SEP-2663-Tasks-Extension.md` *(was missing)*
Pulls the experimental Tasks feature out of core and republishes it as extension
`io.modelcontextprotocol/tasks`, redesigned for the sessionless world: server-directed (server mints
`taskId`), polling via `tasks/get`, new `tasks/update`, removed `tasks/result`/`tasks/list`/`tasks/delete`,
dedicated `tasks/cancel`, 5 states, `ttlMs`/`pollIntervalMs`.

### 5. Deprecate Roots, Sampling, and Logging (SEP-2577) — `SEP-2577-Deprecate-Roots-Sampling-Logging.md` *(was filed as "pending"; it landed)*
Marks all three features `@deprecated` (non-breaking; 12-month support window per SEP-2596). These were
the highest-complexity / lowest-adoption features and the source of the server→client callback surface
MRTR had to work around. Migrations: tool params/resource URIs (Roots), direct LLM APIs (Sampling),
stderr/OTel (Logging).

---

## 🟠 Medium changes

### HTTP Transport Standardization (SEP-2243) — `SEP-2243-HTTP-Headers.md` — **now Status: Final**
Required `Mcp-Method` / `Mcp-Name` headers on Streamable HTTP POSTs; custom `x-mcp-header` routing headers.
**Correction to the existing note:** `x-mcp-header` now applies to `integer`/`string`/`boolean` (**not
`number`**), MAY appear at any nesting depth, and the header-mismatch error code changed `-32001` →
`-32020` (see `Error-Code-Allocation-Renumbering.md`).

### JSON Schema 2020-12 for Tool Schemas (SEP-2106) — `SEP-2106-JSON-Schema-2020-12.md` *(was missing)*
`inputSchema` allows full 2020-12 (composition/`$ref`), `outputSchema` no longer requires `type:"object"`
(arrays/primitives allowed), and `structuredContent` widens from object to `unknown`. Adds `$ref`/no-network
and composition-bounds guardrails.

### TTL for List Results (SEP-2549) — `SEP-2549-TTL-List-Results.md` *(was filed as "pending"; it landed)*
New `CacheableResult` with required `ttlMs` + `cacheScope` (`public`/`private`) on `tools/list`,
`prompts/list`, `resources/list`, `resources/read`, `resources/templates/list`, `server/discover`.
Transport-agnostic freshness hint that complements `listChanged`.

### Feature Lifecycle & Deprecation Policy (SEP-2596) — `SEP-2596-Feature-Lifecycle-Deprecation.md` *(was "pending"; landed)*
Formal Active → Deprecated → Removed states, 12-month minimum window, deprecated-features registry.
Also reclassifies the HTTP+SSE transport and `includeContext: "thisServer"/"allServers"` as Deprecated.

### Extensions Framework (SEP-2133) — `SEP-2133-Extensions.md`
`extensions` field on Client/ServerCapabilities for forward-compatible capability experimentation. Keys
now MUST use a reverse-DNS prefix. This is the mechanism Tasks (2663) now rides on.

### Deterministic `tools/list` Ordering (PR #2516)
Servers SHOULD return tools in a stable order so clients can keep LLM prompts cacheable (prompt-cache
hit rates). Covered in the original summary's "Tool Cache Optimization" section.

---

## 🟢 Smaller changes

### Authorization & security
- **Issuer (`iss`) claim (SEP-2468)** — `SEP-2468-Issuer-Claim.md` *(was "pending"; landed)*. AS SHOULD
  return `iss` (RFC 9207); client MUST validate it — anti-mix-up for multi-AS.
- **DCR deprecated → Client ID Metadata Documents (PR #2858)** — `DCR-Deprecation-CIMD.md` *(was missing)*.
  CIMD (HTTPS-URL `client_id`) is now preferred; DCR is the compatibility fallback.
- **Multi-AS Migration (SEP-2352)** — `SEP-2352-Multi-AS-Migration.md`. Credentials keyed by issuer;
  re-register on AS change. *(Landed as spec text, no standalone `seps/` file.)*
- **Client Application Types (SEP-837)** — `SEP-837-Client-Application-Types.md`. `application_type`
  required in DCR (moot once a client moves to CIMD).
- **Step-Up Authorization (SEP-2350)** — `SEP-2350-Step-Up-Auth.md`. *(Landed as spec text in the draft
  authorization docs, no standalone `seps/` file.)*
- **Refresh Tokens (SEP-2207)** — `SEP-2207-Refresh-Tokens.md`. OIDC refresh-token guidance.

### Errors & schema hygiene
- **Resource-Not-Found error code (SEP-2164)** — `SEP-2164-Resource-Not-Found-Error.md` *(was missing)*.
  `-32002` → `-32602` (Invalid Params); no more empty-`contents` ambiguity.
- **Error-code allocation & renumbering** — `Error-Code-Allocation-Renumbering.md` *(was missing)*.
  `-32000`..`-32019` implementation-defined, `-32020`..`-32099` reserved for MCP; `HeaderMismatch -32020`,
  `MissingRequiredClientCapability -32021`, `UnsupportedProtocolVersion -32022`.

### Observability & request association
- **OpenTelemetry Trace Context (SEP-414)** — `SEP-414-Trace-Context.md`. `traceparent`/`tracestate`/
  `baggage` in `_meta` (DNS-prefix exception).
- **Server Request Association (SEP-2260)** — `SEP-2260-Server-Request-Association.md`. Server requests
  MUST be tied to a client request. *(Now largely subsumed/tightened by MRTR (2322) in practice.)*

### Elicitation
- **Form-mode scope clarification** — the strict `MUST NOT` on sensitive data is limited to credentials/
  tokens (passwords, API keys, payment data); general PII is left to server discretion. Also note MRTR
  **removed** `notifications/elicitation/complete` and the `elicitationId` field.

---

## ⚙️ Governance & process (landed)
- **PR-based SEP workflow (SEP-1850)**, **Contributor Ladder (SEP-2148)**, **Working Groups Charter
  (SEP-2149)**, **Succession & Amendment (SEP-2085)**, **SDK Tiering (SEP-1730)**.
- **Conformance tests required for Final SEPs (SEP-2484)** *(was "pending"; merged)* — Standards-Track
  SEPs need conformance tests before reaching Final.

---

## ⛔ Did NOT land (adjust expectations)
- **Pluggable Transports (SEP-2598)** — `SEP-2598-Pluggable-Transports.md`. **Not merged anywhere** on
  upstream `main` (no `seps/` file, no schema/doc reference). The stateless + `subscriptions/listen` work
  (2575) reshaped the transport story instead. Treat this note as speculative / not adopted.
- Still-open candidates *not* in this draft (re-check each PR's status): 2614 keywords, 2571 resource
  submission, 2564 server-side list filtering, 2557 adapt-tasks, 2532 resource streaming, 2495 event-driven
  tools, 2487 `execution.requirements`, 2433 transfer descriptors, 2419 `cache_hint`, 2417 model
  preferences, 2448 telemetry.

---

## Quick landed/missed ledger (vs. the original review)
| Bucket | Items |
|--------|-------|
| Tracked & correct | 2575, 2567, 2243, 2133, 414, 2260, 2350, 2352, 837, 2207, tool ordering, governance |
| Filed "pending" → actually landed | **2549, 2577, 2596, 2468, 2484** |
| Missing entirely → new notes added | **2322 (MRTR), 2663 (Tasks ext), 2106, 2164, error renumbering, DCR→CIMD** |
| Effort spent but did NOT land | **2598 (Pluggable Transports)** |
