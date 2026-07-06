# Draft Q2 2026: Deprecate Dynamic Client Registration in favor of Client ID Metadata Documents (PR #2858)

## Overview

This draft **deprecates OAuth 2.0 Dynamic Client Registration (DCR, [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591))** as MCP's client-registration mechanism and elevates **Client ID Metadata Documents (CIMD)** to the preferred approach. DCR remains available for backward compatibility with authorization servers that don't support CIMD.

There's no dedicated SEP number (it landed via PR #2858), but it's a substantive auth change that belongs alongside your existing auth notes (SEP-837 application types, SEP-2352 multi-AS migration, SEP-2468 issuer claim) — it changes *how a client gets a `client_id` in the first place*.

## The Core Problem

DCR requires every client to `POST` to each authorization server's `registration_endpoint` to obtain a `client_id`, and it's where the `application_type` redirect-URI conflicts from SEP-837 came from. It's stateful (the AS stores a registration), it doesn't scale well to the "any client can talk to any server with no prior relationship" world MCP is moving toward, and it forces an extra round-trip and server-side storage for what is often an ephemeral client.

## Key Changes

### 1. Registration preference order

The draft now lays out a preference order for obtaining client credentials:

1. **Client ID Metadata Documents** — when client and server have no prior relationship (the most common case).
2. **Pre-registration** — when there's an existing relationship.
3. **Dynamic Client Registration** — for backward compatibility / specific requirements only.

A client SHOULD use CIMD if the AS advertises `client_id_metadata_document_supported` in its metadata, fall back to DCR if the AS advertises a `registration_endpoint`, and only then prompt the user.

### 2. How CIMD works

The client's `client_id` **is an HTTPS URL** pointing to a JSON metadata document the client hosts. The AS fetches that URL to learn about the client instead of the client pre-registering.

- Client `client_id` URL MUST use `https` and contain a path, e.g. `https://example.com/client.json`.
- The document MUST include at least `client_id`, `client_name`, `redirect_uris`, and its `client_id` value MUST match the document URL exactly.
- Clients MAY use `private_key_jwt` for token-endpoint auth with a JWKS.
- Authorization servers SHOULD fetch metadata for URL-formatted `client_id`s, MUST validate the fetched `client_id` matches the URL, MUST validate `redirect_uris` against the document, and SHOULD cache per HTTP cache headers.

## Concrete Example

### Before (DCR)

```http
POST /register HTTP/1.1
Host: auth.example.com
Content-Type: application/json

{ "client_name": "My MCP Client", "redirect_uris": ["https://client.example/cb"],
  "application_type": "native" }
```

→ AS stores a registration and returns a minted `client_id`. Every new client does this against every new AS.

### After (CIMD)

The client just hosts a document and uses its URL as the `client_id` — no registration call:

```json
// Hosted at https://client.example/client.json  (this URL *is* the client_id)
{
  "client_id": "https://client.example/client.json",
  "client_name": "My MCP Client",
  "redirect_uris": ["https://client.example/cb"]
}
```

The authorization request simply references `client_id=https://client.example/client.json`; the AS fetches and validates the document.

## Why this matters

CIMD removes the per-client-per-server registration handshake, which is exactly the friction point for a protocol where arbitrary clients connect to arbitrary servers. It also sidesteps a class of DCR problems (redirect-URI conflicts, registration storage, `application_type` ambiguity). DCR isn't gone — it's the compatibility fallback — but new implementations should build on CIMD.

## Considerations

- CIMD requires the client to **host an HTTPS document**, which is trivial for web/remote apps but awkward for purely local clients — those will still lean on DCR or pre-registration.
- SEP-837's `application_type` guidance still applies while DCR is in use; it becomes moot for clients that move to CIMD.
- Ties into SEP-2468: an AS that fetches a CIMD-based client identity should still return `iss` so the client can validate the authorization response.
