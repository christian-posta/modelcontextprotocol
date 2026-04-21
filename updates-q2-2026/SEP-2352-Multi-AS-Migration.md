# Updates Q2 2026: Multi-Authorization Server Migration & Binding (SEP-2352)

## Overview

SEP-2352 clarifies how MCP clients should manage their credentials when dealing with multiple Authorization Servers (AS), or when an MCP server migrates from one Authorization Server to another. It introduces strict rules around **Authorization Server Binding** to prevent clients from leaking credentials or breaking authorization flows by sending the wrong credentials to the wrong server.

## The Core Problem

An MCP Server's "Protected Resource Metadata" can list multiple valid Authorization Servers (e.g., an enterprise might be migrating from Okta to Auth0, or supporting both simultaneously). 

Before this SEP, if a client (like Claude Desktop) registered itself with Okta and got a `client_id` and `client_secret` via Dynamic Client Registration (DCR), and then the MCP server suddenly told the client to use Auth0 instead, the client might blindly send its Okta `client_id` to Auth0. 

This causes two major issues:
1. **Broken Flows:** Auth0 has no idea what that `client_id` is, leading to immediate 401/400 errors.
2. **Security Risks:** The client is effectively leaking its credentials issued by one entity to a completely different entity.

## Key Changes Introduced by SEP-2352

To solve this, the spec now mandates strict scoping of client state based on the Authorization Server's `issuer` URL.

### 1. Separate Registration State
When multiple Authorization Servers are listed, each is considered an entirely independent OAuth 2.0 entity. 
* Clients **MUST** maintain separate registration state (client credentials, tokens) per authorization server.
* Clients **MUST NOT** assume that credentials valid for one AS will be accepted by another.

### 2. Authorization Server Binding
Clients that use pre-registered credentials or persist credentials obtained via Dynamic Client Registration (DCR) **MUST** bind those credentials to the specific authorization server that issued them, keyed by the AS's `issuer` identifier.
* **Migration Rule:** If the Authorization Server changes (detected via an update in the MCP Server's protected resource metadata), the client **MUST NOT** reuse the old client credentials. It **MUST** perform a new DCR handshake with the new Authorization Server to get new credentials.
* **Pre-registered Rule:** If a client was hardcoded with a specific `client_id` for Okta, and the MCP server switches to Auth0, the client **SHOULD** immediately surface an error to the user rather than silently attempting to use mismatched credentials.

### 3. The CIMD Exception (Portable Client IDs)
There is one major exception: **Client ID Metadata Documents (CIMD)**.
If a client uses an HTTPS URL as its `client_id` (e.g., `client_id=https://app.claude.ai/oauth/metadata.json`), these IDs *are* portable across Authorization Servers. 
Because the Auth Server fetches the metadata dynamically on demand, no re-registration is needed when the MCP server switches from Okta to Auth0. The new Auth Server will simply ping the client's URL to establish trust.

## What Multi-AS Metadata Looks Like

When an MCP server supports multiple Authorization Servers (e.g., during a migration), its Protected Resource Metadata lists them all in the `authorization_servers` array:

```json
GET https://api.internal.com/.well-known/oauth-protected-resource

{
  "resource": "https://api.internal.com",
  "authorization_servers": [
    "https://okta.internal.com",
    "https://auth0.internal.com"
  ],
  "scopes_supported": ["read:data", "write:data"]
}
```

The client must select which AS to use (following RFC 9728 Section 7.6 guidance), and maintain completely separate credential state for each.

## Concrete Example: Handling an Auth Migration

Here is how an MCP Client handles a server migrating its Authorization Provider:

**1. Initial State (Okta):**
The client connects to `https://api.internal.com`. The metadata points to Okta. The client dynamically registers itself, getting `client_id="okta-123"`. It saves this internally:
```json
// Client's Internal State
{
  "https://okta.internal.com": {
    "client_id": "okta-123",
    "access_token": "..."
  }
}
```

**2. The Migration (Auth0):**
A month later, the enterprise migrates. The client attempts to use its token, gets a 401, and fetches the new metadata. The metadata now points to Auth0 (`https://auth0.internal.com`).

**3. The Client's Reaction:**
* **❌ PROHIBITED:** The client tries to send `client_id="okta-123"` to Auth0's `/authorize` endpoint.
* **✅ ALLOWED (DCR):** The client looks at its internal state, sees no entry for `https://auth0.internal.com`, and initiates a brand new Dynamic Client Registration with Auth0 to get a new `client_id="auth0-999"`.
* **✅ ALLOWED (CIMD):** If the client was using CIMD (`client_id="https://claude.ai/mcp.json"`), it completely ignores the migration. It just sends the same URL to Auth0, and Auth0 resolves it dynamically.