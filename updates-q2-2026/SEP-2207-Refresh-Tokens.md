# Updates Q2 2026: Refresh Token Workflows (SEP-2207)

## Overview

SEP-2207 provides guidance for how MCP implementations should handle refresh token issuance and requests, particularly when Authorization Servers support the `offline_access` scope from OpenID Connect (OIDC). The guidance covers both MCP Clients (like Claude, Cursor, VS Code) and MCP Servers acting as OAuth 2.0 Protected Resources.

Before this SEP, users often suffered from frequent re-authentication prompts because clients didn't know how to explicitly ask for long-lived access, and servers didn't know if clients could safely store long-lived tokens.

## The Core Problem

MCP uses OAuth 2.1 for authorization. However, many enterprise deployments use Authorization Servers that also support OIDC. This creates a specific gap regarding the `offline_access` scope (the standard OIDC way to ask for a refresh token):

1. **Clients weren't requesting refresh tokens:** Major MCP clients (Cursor, Claude, VS Code, etc.) weren't explicitly asking for the `offline_access` scope because they didn't know whether the Authorization Server supported, expected, or required it.
2. **Resource servers shouldn't specify `offline_access`:** The `offline_access` scope is not a resource-specific scope — it's a concern between the client and Authorization Server. Including it in `WWW-Authenticate` headers or Protected Resource Metadata is semantically incorrect since it implies the resource *requires* refresh tokens.
3. **Authorization Server inconsistency:** Different Authorization Servers behave differently when issuing refresh tokens, especially when clients don't specify `refresh_token` as a grant type or request `offline_access`.
4. **Interoperability gap:** Without guidance, implementations behave inconsistently, leading to poor user experience (frequent re-authentication) or security issues (issuing refresh tokens to clients that can't securely store them).

## Key Changes Introduced by SEP-2207

To address this, SEP-2207 provides guidance for both Clients and Servers:

### 1. Client Requirements (Scope Augmentation)
* **Advertise Capability:** Clients **SHOULD** include `refresh_token` in their `grant_types` client metadata during registration to indicate they support refresh tokens.
* **Dynamic Scope Augmentation:** When the client desires a refresh token and the Authorization Server metadata contains `offline_access` in its `scopes_supported` field, the client **MAY** add the `offline_access` scope to the list of scopes from the resource server before making authorization requests.
* **No Guarantee:** Clients **MUST NOT** assume that advertising support or requesting `offline_access` guarantees they will receive a refresh token. The Authorization Server retains discretion based on its policies. *(This is the only strict requirement in the SEP.)*

### 2. Server Requirements (Resource Servers)
* MCP Servers **SHOULD NOT** include `offline_access` in the `scope` parameter of `WWW-Authenticate` headers, as refresh tokens are not a resource requirement.
* MCP Servers **SHOULD NOT** include `offline_access` in `scopes_supported` in Protected Resource Metadata, as it is not a resource-specific scope.

## Why this matters

1. **Massively Improved UX:** By standardizing how clients ask for `offline_access`, users will no longer have to constantly re-authenticate with their enterprise SSO providers every few hours. The client can seamlessly refresh the token in the background.
2. **Reduced Token Leakage:** By not issuing refresh tokens to clients that don't advertise `refresh_token` in their `grant_types`, the risk of long-lived tokens being stored insecurely is reduced. Note that since client metadata is self-reported, Authorization Servers **MAY** apply additional restrictions (domain allowlists, reputation checks, verification requirements) rather than solely relying on client metadata claims.
3. **Semantic Correctness:** It properly separates the concerns of the *Resource Server* (which only cares about permissions like `read:database`) from the *Auth Server* (which cares about token lifecycles).

Here is how the interaction should look under the new SEP-2207 guidelines.

### ❌ PROHIBITED: Server Asking for Refresh Tokens
When an AI client tries to read a database but lacks a token, the MCP Server returns a 401 Unauthorized. Before SEP-2207, developers building MCP Servers often wondered if they should include `offline_access` in this 401 response to guarantee their users got a long-lived session (and didn't have to constantly log in). Some implementations mistakenly added it.

**Invalid Server Response:**
```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource",
                         scope="read:database offline_access"
```

**Why is this wrong?**
As detailed in the SEP (referencing OAuth 2.1 Section 5.3.1), the `WWW-Authenticate` header is strictly for the Resource Server to say, *"Here are the permissions required to access this specific data."* The database requires the `read:database` permission to function. It does *not* require a refresh token to function. 

A refresh token (`offline_access`) is purely a lifecycle agreement between the Client (e.g., Claude Desktop) and the Auth Server (e.g., Okta). By forcing `offline_access` into the resource header, the MCP Server was semantically misrepresenting its own security requirements. It also created an anti-pattern: if a lightweight, browser-based client (which has no secure way to store a refresh token) connected to this server, the server would be forcing it to ask for a token it couldn't safely keep.

### ✅ ALLOWED: Client Scope Augmentation
Instead, the server only asks for resource-specific scopes. The *Client* is the one that decides to ask the Auth Server for a refresh token.

**1. Valid Server Response:**
```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource",
                         scope="read:database"
```

**2. Client discovers AS supports `offline_access`:**
Before augmenting, the client fetches the Authorization Server's metadata and checks whether `offline_access` appears in `scopes_supported`:
```json
GET https://auth.example.com/.well-known/oauth-authorization-server

{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "scopes_supported": ["read:database", "write:database", "offline_access"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  ...
}
```
The client sees `"offline_access"` in `scopes_supported` — this means the AS recognizes the OIDC convention. Because the client is a desktop app capable of securely storing secrets, it decides to augment.

**3. Client Authorization Request (Augmented):**
The client dynamically appends `offline_access` to the scopes from the resource server's 401 before opening the user's browser:
```http
GET /authorize?
  response_type=code&
  client_id=claude-desktop&
  scope=read:database offline_access&
  redirect_uri=...
```

If the AS metadata had *not* included `offline_access` in `scopes_supported`, the client would skip augmentation and send only `scope=read:database`. The AS might still issue a refresh token based on its own policies and the client's `grant_types` metadata — but the client **MUST NOT** assume this will happen.

## How Clients Advertise Support (CIMD Example)

Because the Authorization Server must decide whether it is safe to issue a refresh token, the client must explicitly declare that it wants them and knows how to handle them. 

For clients that use **Client ID Metadata Documents (CIMD)** (a method where the client hosts a JSON file containing its own OAuth configuration), the client advertises this support by adding `refresh_token` to its `grant_types` array.

Here is an example of an MCP Client's metadata document (`https://app.example.com/oauth/metadata.json`):

```json
{
  "client_id": "https://app.example.com/oauth/metadata.json",
  "client_name": "Example AI Client",
  "client_uri": "https://app.example.com",
  "redirect_uris": [
    "http://127.0.0.1:3000/callback"
  ],
  "grant_types": [
    "authorization_code",
    "refresh_token" 
  ],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

Notice `token_endpoint_auth_method: "none"`. This explicitly flags the client as a **Public Client** (meaning it has no `client_secret`). Because `refresh_token` is listed in the `grant_types`, the Auth Server sees this document and knows: *"This is a public client requesting refresh tokens. I must ensure Refresh Token Rotation is enforced before I issue one."*

## Security Implications: Public Clients & PKCE

The separation of concerns in SEP-2207 is complemented by the broader MCP auth spec's handling of **OAuth Public Clients**.

Many MCP clients (like CLI tools or Desktop apps) are considered "Public Clients." Unlike a backend server, they cannot be trusted with a `client_secret` because the secret would have to be shipped in the source code or binary, where an attacker could easily extract it.

To secure the initial login without a secret, these clients use **PKCE** (Proof Key for Code Exchange). The MCP auth spec requires all MCP clients **MUST** implement PKCE (per OAuth 2.1 Section 7.5.2). PKCE proves that the client who *started* the login flow is the exact same client who is *finishing* it. However, once the login is done and the client receives a Refresh Token, a new problem emerges: **How do you secure a Refresh Token if the client has no secret to authenticate itself later?**

If a refresh token is stolen from a public client, an attacker could theoretically use it forever.

### The Mitigation: Token Rotation
The MCP auth spec (separate from SEP-2207) requires that for public clients, authorization servers **MUST** rotate refresh tokens as described in OAuth 2.1 Section 4.3.1.

When a public MCP client uses a refresh token to get a new access token, the Auth Server invalidates the old refresh token and issues a *brand new one*. 
* If an attacker steals a refresh token and uses it, the legitimate user's client will eventually try to use its copy of that same token. 
* The Auth Server will see a previously invalidated token being used again, realize a theft has occurred, and revoke the refresh token grant so no *new* tokens can be issued.
* However, any access token the attacker already obtained remains valid until it expires — access tokens are typically self-contained (e.g., JWTs) and validated locally by resource servers without calling back to the AS. This is why the MCP auth spec also recommends that Authorization Servers **SHOULD** issue short-lived access tokens to limit the window of damage.

This is why the separation introduced by SEP-2207 matters: by keeping `offline_access` out of the resource server's `WWW-Authenticate` header, the Auth Server retains the right to look at the client making the request, determine if it is a public client, ensure Refresh Token Rotation is enabled for that client, and only *then* decide whether to issue a refresh token.