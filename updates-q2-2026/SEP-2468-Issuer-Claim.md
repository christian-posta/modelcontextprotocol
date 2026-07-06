# Draft Q2 2026: Recommend the Issuer (`iss`) Claim in Auth Responses (SEP-2468)

## Overview

SEP-2468 hardens MCP's OAuth flow against authorization-code mix-up attacks by adopting [RFC 9207](https://datatracker.ietf.org/doc/html/rfc9207): authorization servers **SHOULD** include the `iss` (issuer) parameter in their authorization responses, and MCP clients **MUST** validate a present `iss` against the recorded issuer before redeeming the authorization code.

This slots directly next to your existing auth notes (SEP-2352 multi-AS migration, SEP-837 application types) — it's another piece of tightening what happens when a client talks to more than one authorization server. Landed as a **Minor** change; previously on your "pending" list.

Status: **Final**, Standards Track. Author: Peter Alexander.

## The Core Problem

When a client can talk to multiple authorization servers, an attacker who controls (or can trick the client into using) one AS can try a **mix-up attack**: intercept an authorization response and feed the code back to the client as if it came from a *different*, more privileged AS. Without an issuer identifier in the response, the client has no reliable way to detect that the code it's about to redeem originated from the wrong server.

## Key Changes Introduced by SEP-2468

1. **AS behavior:** authorization servers **SHOULD** return the `iss` parameter (per RFC 9207) in authorization responses.
2. **Client behavior:** if `iss` is present, the MCP client **MUST** validate it against the issuer it recorded for that authorization request, before exchanging the code for a token. A mismatch means abort.

## Concrete Example

### Before

```http
# Authorization response redirect — no issuer, client can't detect mix-up
HTTP/1.1 302 Found
Location: https://client.example/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
```

The client just redeems `code` at whichever token endpoint it thinks it's talking to.

### After

```http
# RFC 9207: issuer echoed in the response
HTTP/1.1 302 Found
Location: https://client.example/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz&iss=https%3A%2F%2Fauth.example.com
```

The client compares `iss` (`https://auth.example.com`) against the issuer it associated with this `state`/request. If they don't match, it refuses to redeem the code.

## Why this matters

It's a small, standards-aligned change with an outsized security payoff in exactly the multi-AS world SEP-2352 is enabling. As MCP clients increasingly bind credentials per-issuer and migrate between authorization servers, an unambiguous issuer signal on every authorization response is what makes those bindings enforceable.

## Considerations

- It's `SHOULD` on the server side, so clients still have to handle responses **without** `iss` (older/again non-compliant AS) — the `MUST` only bites when `iss` is actually present.
- Pairs naturally with SEP-2352's rule that credentials are keyed by issuer: `iss` validation is how you catch a response that doesn't belong to the issuer you keyed against.
