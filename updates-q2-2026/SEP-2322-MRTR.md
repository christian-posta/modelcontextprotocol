# Draft Q2 2026: Multi Round-Trip Requests / MRTR (SEP-2322)

## Overview

SEP-2322 is arguably the single largest behavioral change in this draft. It introduces the **Multi Round-Trip Requests (MRTR)** pattern, which completely replaces the old mechanism for **server-initiated requests** — the way a server used to "call back" to the client mid-tool-call to ask for sampling (`sampling/createMessage`), roots (`roots/list`), or elicitation (`elicitation/create`).

It is the connective tissue that makes SEP-2575 (Stateless MCP) and SEP-2567 (Sessionless MCP) actually workable. You cannot have a server pause a tool call and send a fresh request *back down a connection* to the client if there is no durable connection and no session to hang that request off of. MRTR turns those server→client callbacks into ordinary request/response round-trips that any stateless backend can service.

Status: **Final**, Standards Track. Authors: Mark D. Roth, Caitie McCaffrey, Gabriel Zimmerman.

## The Core Problem

Under the previous spec, if a tool needed extra information mid-execution (e.g. it needed the model to summarize something via sampling, or needed the user to confirm a value via elicitation), the **server** would open a new JSON-RPC request *in the reverse direction* — server to client — while the original `tools/call` was still pending.

This required three things that are toxic to horizontal scale:

1. **A persistent bidirectional channel.** The server had to hold the original request open (typically over an SSE stream) and push a new request down it. No connection, no callback.
2. **Sticky routing.** The callback and the eventual answer had to land on the *exact same server instance* that was mid-way through the original tool call, because that instance held the in-flight execution state in memory.
3. **Shared/observable state across instances.** Any load-balanced deployment had to replicate the pending-request state, or pin the client to one box.

For the common case — an ephemeral, stateless tool running behind a round-robin load balancer — this was the thing that made "just scale it horizontally" impossible.

## Key Changes Introduced by SEP-2322

### 1. Server-initiated requests become an "input required" result

Instead of the server sending a new request to the client, the server **responds to the client's original request** with a special result that says *"I can't finish yet — I need you to go get these things for me."*

- New result type `InputRequiredResult` carrying an `inputRequests` map and an opaque `requestState` string.
- The requests the server wants fulfilled (`sampling/createMessage`, `elicitation/create`, `roots/list`) are packaged **as data inside the result**, keyed by a server-assigned string ID.

### 2. A new required `resultType` discriminator on *every* result

This is a schema-wide change, not just an MRTR feature. `Result.resultType` is now **required** on all results:

- `"complete"` — an ordinary, finished result.
- `"input_required"` — the interim MRTR result described above.
- `"task"` — a task handle (see SEP-2663, Tasks Extension).
- Extensions MAY define additional values.

Backward-compat rule: a result from an older server that **omits** `resultType` MUST be treated as `"complete"`.

### 3. The client fulfills the requests, then *retries the original request*

There is no callback to answer. The client gathers the requested inputs locally (prompts the user, runs sampling against its own LLM, enumerates roots), then **re-issues the original request** with:

- an `inputResponses` map keyed back to the same IDs, and
- the `requestState` blob echoed back **verbatim**.

The retry MUST use a **new JSON-RPC `id`** (it is a new request on the wire).

### 4. `requestState` — opaque, server-owned continuation

`requestState` is a base64-ish opaque token the server uses to resume where it left off (it can encode the tool arguments, a step counter, etc.). Rules:

- The client MUST echo it back unchanged and MUST NOT inspect or depend on it.
- The server MUST treat it as untrusted input and validate it; it SHOULD sign/encrypt it if tampering is a concern, and MUST cryptographically bind any user-specific state to the authenticated user.

### Where does the state actually live? (the key insight)

A natural objection: "if there's a `requestState`, there's state — so if the load balancer sends the retry to a *different* instance, how does that instance know anything about it?" The answer is the whole point of the design: **the state doesn't live on any instance. It travels with the client.**

Think of `requestState` as the **inverse of a session ID**:

- A **session ID** (like the now-removed `Mcp-Session-Id`) is a *pointer* to state that lives on the server. An instance holding only the pointer knows nothing — which is exactly why the old model needed sticky routing or a shared Redis/Postgres to bridge instances.
- **`requestState` is the state itself**, serialized into an opaque blob and handed to the client to carry. The receiving instance doesn't *look it up* — it *decodes it* out of the request that just arrived. Everything needed to resume came in on the wire.

It's the same technique as a JWT, an encrypted cookie, or continuation-passing style: push the state out to the caller so the server can stay amnesiac. The SEP says so directly — "the request state is sent to the client which echoes back the state to the server, **allowing the server to remain stateless**," and its worked example accumulates context in `requestState` across rounds "so that the final update can be executed **without any server-side storage**." The motivation section explicitly frames a shared storage layer and stateful (sticky) load balancing as the *expensive alternatives MRTR exists to eliminate* — not as something you're expected to add.

**So how does a different instance resume?** In Step 2 below, Instance A encodes what it needs (tool arguments, which step it's on, any partial results) into `requestState` and returns it. In Step 3 the client re-sends the request — with `inputResponses` + that `requestState` — and the load balancer can drop it on Instance B. Instance B decodes the blob, reconstructs the full context, and continues. It never talked to Instance A.

**Two consequences fall out of client-carried state:**

1. **This is *why* the security rules exist** (see Challenge #2). Once the continuation lives in an untrusted client's hands, the server MUST sign/encrypt it and bind it to the authenticated user — otherwise a client could tamper with it or replay someone else's continuation. With server-side state you got that protection for free; with a client-carried token you have to add it.
2. **When the blob gets too big or the work is genuinely long-lived**, self-contained encoding stops being practical — you don't want megabytes of partial state round-tripping every turn. *That* is the line where you switch to the **persistent / Tasks flow** (below): there the state legitimately does live server-side under a `taskId`, and the `Mcp-Name: <taskId>` routing header sends you back to the instance/store that holds it. MRTR gives you both: a stateless token for the cheap ephemeral case, a server-side task for the heavy case.

> Nuance: the spec only requires that `requestState` be *opaque to the client* — it doesn't strictly forbid a server from putting a lookup key in it and going back to a shared store. But doing so re-introduces exactly the shared-storage cost MRTR was built to avoid; the design intent, and every example in the SEP, is self-contained encoding.

### What's inside `requestState` — and how to encode it

The wire type is just `string`, and the client treats it as an opaque blob — so the server is free to choose the format. It's worth being concrete about both *what* goes in and *how* it's protected, because the two decisions are independent.

**The logical payload** is whatever the tool needs to pick up where it left off: the original arguments, a step/phase marker, anything already gathered in earlier rounds, and (critically) the identity the state is bound to. For the `open_pr` tool from the example above, mid-flow it might be:

```json
{
  "tool": "open_pr",
  "args": { "repo": "acme/api" },
  "phase": "awaiting_github_login",
  "gathered": {},
  "sub": "user_12345",
  "iat": 1751731200,
  "exp": 1751731500
}
```

Then you pick an encoding tier based on how sensitive that payload is and how much you distrust the client. **These are the same three tiers HTTP cookies went through — and the reasoning is identical.**

**Tier 1 — Plain `base64(JSON)` (illustration only).** This is literally what the example blob in this doc is: decode `eyJsb2NhdGlvbiI6Ik5ldyBZb3JrIn0` and you get `{"location":"New York"}`. It's readable *and forgeable* by the client, with no integrity or confidentiality.

```jsonc
// requestState = base64url(JSON) — fine for a spec example, almost never fine in production
"eyJ0b29sIjoib3Blbl9wciIsImFyZ3MiOnsicmVwbyI6ImFjbWUvYXBpIn0sInBoYXNlIjoiYXdhaXRpbmdfZ2l0aHViX2xvZ2luIn0"
```

Only acceptable if the state is fully non-sensitive **and** you genuinely don't care whether the client tampers with it — which, once you include `sub` for user-binding, is basically never. The SEP's `MUST validate` / `MUST bind to the authenticated user` rules effectively rule this out for real state.

**Tier 2 — Signed (HMAC / JWS / a JWT).** Add a signature so the server can detect any tampering; the payload stays *visible* to the client (base64 is not encryption) but becomes unforgeable. Use this when the contents aren't secret but must not be altered or fabricated.

```jsonc
// A JWT is the off-the-shelf version: header.payload.signature
"eyJhbGciOiJIUzI1NiIsImtpZCI6IjIwMjYtMDcifQ.eyJ0b29sIjoib3Blbl9wciIsInN1YiI6InVzZXJfMTIzNDUiLCJleHAiOjE3NTE3MzE1MDB9.3n2v...HMAC..."
//  \__ header (alg, key id) __/ \__________ payload (visible!) __________/ \__ HMAC signature __/
```

On the retry, the server recomputes the signature with its secret; a mismatch means reject. The `kid` lets you rotate keys; `exp` bounds how long a captured blob is replayable.

**Tier 3 — Encrypted + authenticated (AEAD / JWE).** When the payload holds anything the client shouldn't see — internal IDs, partial results, business logic hints — encrypt it with an authenticated cipher (e.g. AES-GCM, XChaCha20-Poly1305, or a JWE). This gives confidentiality *and* integrity in one step.

```jsonc
// Opaque ciphertext — the client can carry it but can't read or forge it
"v1.k2026-07.uT9c1o8v...nonce...+ciphertext+authtag..."
```

Bind it to the user by putting `sub` inside the encrypted payload, or by passing the authenticated user as **additional authenticated data (AAD)** to the AEAD so decryption fails if the blob is replayed under a different identity.

**Practical concerns regardless of tier:**

- **User binding is mandatory, not optional.** The `sub` (or AAD) is what stops a client from swapping in another user's continuation. Verify on every retry that the bound identity equals the currently authenticated caller.
- **Set an expiry.** `exp` (or a server-side max-age check) bounds the replay window for a leaked blob and gives you a natural "this multi-round-trip took too long, start over" signal.
- **Mind the size.** The blob rides in the request body (and could be large after several rounds of accumulated context). If it's growing without bound, that's the signal to switch to the persistent/Tasks flow rather than fatten `requestState` every turn.
- **Version and rotate keys.** A leading version tag / `kid` lets you change format or roll signing/encryption keys without breaking in-flight round-trips.

### 5. MRTR is only allowed on a few request types

`InputRequiredResult` MAY be returned only from `tools/call`, `prompts/get`, `resources/read`, and (for the persistent flow) `tasks/result`. It MUST NOT be returned from any other request (`ping`, list methods, etc.). This builds on and further tightens **SEP-2260** (server requests must be associated with a client request).

## Concrete Example: Sampling + Elicitation mid tool-call

### Old Server-Initiated Model

The client calls `tools/call`. The server, mid-execution, opens **new requests back to the client** over the held-open stream:

```jsonc
// Server -> Client (a brand new request, reverse direction), while tools/call is pending
{ "jsonrpc": "2.0", "id": "srv-1", "method": "elicitation/create",
  "params": { "message": "Please provide your GitHub username", "requestedSchema": { /* ... */ } } }

// Server -> Client (another reverse-direction request)
{ "jsonrpc": "2.0", "id": "srv-2", "method": "sampling/createMessage",
  "params": { "messages": [ /* ... */ ], "maxTokens": 100 } }
```

The client answers each, the server correlates them back to the in-memory tool call, and only then returns the final `tools/call` result. All of this required the same instance + a live stream.

### New MRTR Model

**Step 1 — Client calls the tool normally** (self-contained, per SEP-2575):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": { "name": "open_pr", "arguments": { "repo": "acme/api" } }
}
```

**Step 2 — Server responds with `input_required`** (not a callback — a *result*):

```jsonc
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "github_login": {
        "method": "elicitation/create",
        "params": {
          "message": "Please provide your GitHub username",
          "requestedSchema": { "type": "object",
            "properties": { "name": { "type": "string" } }, "required": ["name"] }
        }
      },
      "capital_of_france": {
        "method": "sampling/createMessage",
        "params": { "messages": [ { "role": "user",
          "content": { "type": "text", "text": "What is the capital of France?" } } ],
          "maxTokens": 100 }
      }
    },
    "requestState": "eyJsb2NhdGlvbiI6Ik5ldyBZb3JrIn0"
  }
}
```

**Step 3 — Client fulfills locally and *retries the original tool call*** with a **new id**, attaching `inputResponses` and echoing `requestState`:

```jsonc
{
  "jsonrpc": "2.0",
  "id": 2,                          // NEW id — this is a fresh request
  "method": "tools/call",
  "params": {
    "name": "open_pr",
    "arguments": { "repo": "acme/api" },
    "inputResponses": {
      "github_login": { "action": "accept", "content": { "name": "octocat" } },
      "capital_of_france": {
        "role": "assistant",
        "content": { "type": "text", "text": "The capital of France is Paris." },
        "model": "claude-3-sonnet-20240307", "stopReason": "endTurn"
      }
    },
    "requestState": "eyJsb2NhdGlvbiI6Ik5ldyBZb3JrIn0"   // echoed verbatim
  }
}
```

**Step 4 — Server resumes from `requestState` and returns the real result** (`resultType: "complete"`). Any instance behind the load balancer can service Step 3, because everything it needs is in the request.

## Two flavors: ephemeral vs persistent

- **Ephemeral (stateless):** the flow above. The original request *terminates* at Step 2; the client re-issues it. State lives entirely in `requestState`.
- **Persistent (Tasks):** for long-running work, the server instead puts a **task** into `input_required` status; the client polls `tasks/get`, retrieves the `inputRequests` via `tasks/result`, and answers with the new `tasks/input_response` method. The request does not terminate; state lives server-side in the task. You can transition ephemeral → persistent, but not the reverse.

## Downstream schema fallout you should note

- **`notifications/elicitation/complete` is removed**, and the `elicitationId` field of URL-mode elicitation is removed. Under MRTR the client learns the outcome by retrying, so a server-pushed "complete" signal and its correlation ID no longer fit. Servers that need to correlate an elicitation across retries encode their own identifier inside `requestState`.
- The sampling/roots/elicit request+result types are no longer standalone `JSONRPCRequest`/`Result` members — they become members of the `InputRequest`/`InputResponse` unions.
- The `ServerRequest` union is effectively gone: the server no longer initiates requests as its own top-level messages.

## Why this matters

MRTR is what lets "make MCP stateless" be more than a slogan. Sampling, elicitation, and roots were the three features that *forced* a bidirectional, sticky, stateful connection. By reframing them as "the server returns a to-do list, the client does the work and retries," every request — even one that needs human input or model calls in the middle — becomes a plain, independently-routable round-trip. It is also a net security improvement: there are no longer unsolicited server→client requests arriving out of band (the SEP-2260 concern), because every server ask is a direct, inspectable response to something the client explicitly initiated.

## Challenges for Existing Servers

1. **Tool logic must become resumable.** A server can no longer `await elicitation(...)` inline in the middle of a function. It must be able to stop, serialize its progress into `requestState`, and resume when the retry arrives. This is a real refactor for servers that were written as straight-line async code, and SDKs are expected to deprecate the old inline-await elicitation/sampling helpers.
2. **`requestState` is a security surface.** Because it round-trips through an untrusted client, servers MUST validate it and MUST bind user-specific data to the authenticated identity — otherwise a client could swap in someone else's continuation. Signing/encryption is strongly recommended.
3. **Extra round-trips.** The ephemeral flow re-sends the original request (with its arguments) on every input cycle, which is more bytes on the wire than a single held-open stream. For heavy multi-step interactions the persistent (Tasks) flow is the better fit.
4. **Client-side capability tracking.** The client must be prepared for *any* eligible request (`tools/call`, `prompts/get`, `resources/read`) to come back as `input_required` and know how to fulfill each `InputRequest` type it declared support for.
