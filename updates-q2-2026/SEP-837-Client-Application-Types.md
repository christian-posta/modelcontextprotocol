# Updates Q2 2026: Client Application Types (SEP-837)

## Overview

SEP-837 clarifies requirements for MCP Clients performing Dynamic Client Registration (DCR) with Authorization Servers that support OpenID Connect (OIDC). It explicitly requires clients to declare their `application_type` to prevent registration failures caused by strict Redirect URI validation rules.

## The Core Problem

When an MCP client (like VS Code, Cursor, or a CLI tool) registers itself dynamically with an Authorization Server to get a `client_id`, it provides a list of `redirect_uris` so the Auth Server knows where to send the user after a successful login.

Under the **OpenID Connect Dynamic Client Registration** specification, there is an optional `application_type` parameter. If a client omits this parameter, the OIDC specification mandates that it defaults to `"web"`.

This default behavior created a massive interoperability issue for MCP:
1. **The "Web" Constraint:** Auth Servers expect `"web"` applications to be remote, server-hosted web apps. Therefore, they strictly require HTTPS redirect URIs (e.g., `https://example.com/callback`) and aggressively reject `localhost`, `127.0.0.1`, or custom URL schemes.
2. **The MCP Reality:** Most MCP clients are local desktop apps or CLI tools. They rely on local loopback addresses (`http://127.0.0.1:3000/callback`) or custom URI schemes (like `vscode://mcp/callback`) to receive the OAuth code. 
3. **The Failure:** Because clients weren't explicitly defining their type, Auth Servers assumed they were `"web"` apps and immediately rejected their local loopback `redirect_uris` as insecure, causing the entire connection flow to fail.

## Key Changes Introduced by SEP-837

To resolve this conflict, SEP-837 introduces the following strict guidelines:

1. **Mandatory Declaration:** MCP clients **MUST** specify an appropriate `application_type` during Dynamic Client Registration. (Non-OIDC OAuth servers will simply safely ignore this parameter).
2. **Categorization:**
   * **`"native"`:** Desktop applications, mobile apps, CLI tools, and locally-hosted web applications accessed via `localhost` **SHOULD** use `application_type: "native"`. This tells the Auth Server to allow `127.0.0.1` and custom URI schemes.
   * **`"web"`:** Only remote browser-based applications served from a non-local host **SHOULD** use `application_type: "web"`.
3. **Error Handling:** Clients **MUST** be prepared to handle registration failures due to redirect URI constraints. If rejected, clients **SHOULD** surface a meaningful error to the user or developer. Clients **MAY** retry registration with an adjusted `application_type` or with redirect URIs that conform to the authorization server's requirements.

## Concrete Examples

Here is how an MCP Client's Dynamic Client Registration payload should look under the new guidance.

### ❌ PROHIBITED: Omitting the Application Type
A desktop AI client attempts to register itself but forgets to specify what type of app it is. The OIDC Auth Server defaults it to `"web"`, sees the local IP, and rejects the registration.

**Client Request:**
```http
POST /register HTTP/1.1
Content-Type: application/json

{
  "client_name": "My Desktop AI",
  "redirect_uris": [
    "http://127.0.0.1:8080/oauth/callback",
    "myapp://auth"
  ],
  "grant_types": ["authorization_code"]
}
```

**Auth Server Response:**
```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "invalid_redirect_uri",
  "error_description": "Web applications must use HTTPS redirect URIs. Localhost and custom schemes are not permitted."
}
```

### ✅ ALLOWED: Explicit "Native" Declaration
The client explicitly declares itself as a native application, signaling to the Auth Server that loopback addresses and custom schemes are safe and expected.

**Client Request:**
```http
POST /register HTTP/1.1
Content-Type: application/json

{
  "client_name": "My Desktop AI",
  "application_type": "native",
  "redirect_uris": [
    "http://127.0.0.1:8080/oauth/callback",
    "myapp://auth"
  ],
  "grant_types": ["authorization_code"]
}
```

**Auth Server Response:**
```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "client_id": "auto-generated-client-123",
  "client_secret": "...",
  "application_type": "native",
  "redirect_uris": [
    "http://127.0.0.1:8080/oauth/callback",
    "myapp://auth"
  ]
}
```

## Why this matters

This small metadata addition ensures massive compatibility across the enterprise ecosystem. Major IDEs (like VS Code) and desktop agents rely heavily on OIDC-compliant Auth Servers like Okta, Auth0, or Entra ID. By forcing the `native` declaration, these tools can seamlessly register their local loopback interfaces without constantly tripping the Auth Server's security validation rules.