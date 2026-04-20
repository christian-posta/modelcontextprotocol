# Extension: ext-auth (Authorization Extensions)

**Repository:** [modelcontextprotocol/ext-auth](https://github.com/modelcontextprotocol/ext-auth)

## Overview

`ext-auth` contains official MCP extensions that add authorization capabilities beyond the core MCP specification (which primarily relies on the standard interactive OAuth 2.0 authorization code flow). These extensions are designed to handle specific real-world scenarios where interactive user consent flows are either impossible or inappropriate.

## Key Capabilities & Negotiation Strings

The `ext-auth` repo contains two distinct extensions. Rather than negotiating a generic "auth" extension, clients and servers negotiate these specific mechanisms using the `{vendor-prefix}/{extension-name}` format defined in SEP-2133.

1. **OAuth Client Credentials**
   - **Capability String:** `"io.modelcontextprotocol/oauth-client-credentials": {}`
   - **Use Case:** Machine-to-machine integrations (background services, daemons, CI/CD pipelines, server-to-server APIs).
   - **How it works:** Authenticates without interactive user consent flows, allowing automated systems to securely access MCP tools and resources.

2. **Enterprise-Managed Authorization**
   - **Capability String:** `"io.modelcontextprotocol/enterprise-managed-authorization": {}`
   - **Use Case:** Enterprise environments with centralized Identity Providers (IdPs).
   - **How it works:** Centralizes access control via enterprise IdPs, so employees can access MCP servers through their organization's IdP, enforcing company-wide policies without requiring employees to individually authorize each MCP server.

## Architecture & Support

- These extensions use the standard MCP extension negotiation mechanism. Clients and servers declare their support in the `extensions` field of their capabilities during initialization using the specific capability strings listed above.
- Support varies by client. Both extensions require explicit support from the MCP client and are never active by default.
