# Updates Q2 2026: HTTP Transport Standardization (SEP-2243)

## Overview

SEP-2243 standardizes how routing and context information is exposed over the Streamable HTTP transport. Historically, all routing information (like tool names, methods, or region tags) was buried deep within the JSON-RPC payload. This meant network intermediaries like load balancers, API gateways, and WAFs had to terminate TLS and perform deep packet inspection to route traffic.

This SEP resolves that friction by mirroring critical fields from the JSON payload directly into standard HTTP headers.

## Key Changes Introduced by SEP-2243

### 1. Required Standard Headers

All Streamable HTTP `POST` requests (for both requests and notifications) must now extract specific JSON-RPC fields and append them as headers:

- **`Mcp-Method`**: Mirrors the JSON-RPC `method` field.
- **`Mcp-Name`**: Mirrors `params.name` or `params.uri` (Required for `tools/call`, `resources/read`, and `prompts/get`).

#### Concrete Example: Standard Headers

If the client wants to execute the `get_weather` tool, it generates this JSON-RPC body:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "city": "Seattle" }
  }
}
```

The resulting HTTP request MUST now look like this:

```http
POST /mcp HTTP/1.1
Host: api.example.com
Content-Type: application/json
Mcp-Method: tools/call
Mcp-Name: get_weather

{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"get_weather","arguments":{"city":"Seattle"}}}
```

---

### 2. Custom Tool Parameter Headers (`x-mcp-header`)

Servers can now instruct clients to extract specific tool parameter values and place them into HTTP headers. This is done by adding the `x-mcp-header` extension property to a parameter's definition within the tool's JSON `inputSchema`.

- **Format:** The `x-mcp-header` property defines the suffix for the header. The client MUST prefix this value with `Mcp-Param-`. (e.g., if the schema defines `"x-mcp-header": "Region"`, the generated header becomes `Mcp-Param-Region`).
- **Use Case:** Allows load balancers to route a `tools/call` request to a specific geographic region or tenant cluster based solely on the HTTP header.
- **Constraints:** This property can _only_ be applied to primitive types (number, string, boolean). If a server applies it to a complex object or array, the client MUST reject that tool during the `tools/list` initialization.

#### Concrete Example: Custom `x-mcp-header`

**Server Tool Definition:**

```json
{
  "name": "deploy_server",
  "description": "Deploys a new instance in the target region",
  "inputSchema": {
    "type": "object",
    "properties": {
      "region": {
        "type": "string",
        "description": "The deployment region",
        "x-mcp-header": "Region"
      },
      "instance_type": {
        "type": "string",
        "description": "EC2 Instance Size"
      }
    },
    "required": ["region", "instance_type"]
  }
}
```

**Client HTTP Request:**
Notice how the LLM generates arguments for _both_ `region` and `instance_type`, but the client _only_ turns `region` into an HTTP header because it was specifically tagged with `"x-mcp-header": "Region"` in the schema.

```http
POST /mcp HTTP/1.1
Host: api.example.com
Content-Type: application/json
Mcp-Method: tools/call
Mcp-Name: deploy_server
Mcp-Param-Region: us-east-1

{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"deploy_server","arguments":{"region":"us-east-1", "instance_type": "t3.large"}}}
```

---

### 3. Strict Validation & Security

Because headers and the JSON body could potentially mismatch (leading to routing spoofing attacks), the SEP introduces strict validation rules. Any server processing the message body **MUST** validate that the HTTP header values exactly match the corresponding values in the JSON-RPC body.

#### Concrete Example: Header Mismatch Attack

A malicious user tries to bypass a firewall by putting a safe region in the header (`us-east-1`), but asking the internal server to execute in a restricted region (`us-secret-1`) in the JSON body:

**Malicious HTTP Request:**

```http
POST /mcp HTTP/1.1
Mcp-Method: tools/call
Mcp-Name: deploy_server
Mcp-Param-Region: us-east-1

{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"deploy_server","arguments":{"region":"us-secret-1"}}}
```

**Server HTTP Response (Rejected):**

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 3,
  "error": {
    "code": -32001,
    "message": "HeaderMismatch: HTTP headers do not match request body parameters"
  }
}
```

---

### 4. Value Encoding Rules

To prevent header injection attacks (like injecting `\r\n` characters to add fake headers), clients must safely encode parameter values before placing them in HTTP headers:

- **Standard Types:** Booleans, integers, and standard ASCII strings are passed as-is (e.g., `Mcp-Param-Count: 42`).
- **Base64 Encoding:** If a value contains non-ASCII characters, newlines, carriage returns, or leading/trailing whitespace, the client **MUST** encode it using a specific Base64 format: `=?base64?{encoded_value}?=`.

#### Concrete Example: Base64 Encoding

Imagine a tool that takes a `message` parameter marked with `x-mcp-header: true`. The user inputs a message with a newline: `"Line1\nLine2"`.

**Client HTTP Request:**

```http
POST /mcp HTTP/1.1
Mcp-Method: tools/call
Mcp-Name: send_alert
Mcp-Param-Message: =?base64?TGluZTEKTGluZTI=?=

{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"send_alert","arguments":{"message":"Line1\nLine2"}}}
```

_(Note: `TGluZTEKTGluZTI=` is the base64 encoding of `"Line1\nLine2"`)_

## Why this matters

This standardization is crucial for enterprise deployments. By lifting routing data out of the JSON body and into HTTP headers, standard network infrastructure can now natively route, rate-limit, and monitor MCP traffic using existing, highly optimized tooling without needing to parse complex JSON payloads on every request.

## Challenges for Existing Servers

Implementing this SEP introduces several operational and architectural challenges:

1. **The Validation Burden:** The most significant breaking change is the strict validation mandate. Existing servers must add middleware to intercept the HTTP request, parse both headers and JSON, and perform a deep comparison before execution. Failure to implement this correctly exposes the server to the Header Mismatch Attacks described above.
2. **Unwrapping Encoded Headers:** While every major programming language natively supports Base64 decoding, standard HTTP frameworks (like Express, FastAPI, or Spring) treat incoming headers as plain strings. They will not automatically recognize or unwrap the custom `=?base64?...?=` wrapper format specified by SEP-2243. This encoding is strictly necessary because raw HTTP headers cannot contain newlines (which would break the HTTP protocol and cause Header Injection vulnerabilities) and traditionally struggle with raw Unicode. Therefore, server authors or SDK maintainers must write custom middleware to manually detect this wrapper (e.g., checking if the header string starts with `=?base64?`), slice out the inner payload, and *then* pass it to the language's standard Base64 decoder before validating it against the JSON body.
3. **Protocol Agnosticism vs. HTTP Coupling:** MCP is designed to be transport-agnostic (supporting both HTTP and local `stdio`). The `x-mcp-header` annotation in a tool's `inputSchema` is HTTP-specific metadata — on stdio transports, clients simply ignore it and pass parameters normally in the JSON body. The tool schema itself remains transport-neutral, but server authors should be aware that the header-based routing benefits only apply to HTTP deployments.

## The Asynchronous Upgrade Path

Because MCP involves independent actors (Clients, Servers, and Network Intermediaries), rolling out this SEP requires a phased, asynchronous approach:

### Phase 1: Client Updates (Defensive Rollout)
Clients must be updated first. Updated MCP SDKs automatically extract `method` and `name` to append the mandatory `Mcp-Method` and `Mcp-Name` headers. Older servers will simply ignore these unknown headers and continue processing the JSON body, ensuring backward compatibility.

### Phase 2: Server "Opt-In" (The Friction Point)
Servers begin adopting the new SDK middleware to handle validation and start utilizing `x-mcp-header` in their schemas.
*Note:* Older clients that do not support `x-mcp-header` will still function — they simply won't send the custom headers, so the routing benefits are lost. However, if the server's validation middleware strictly enforces custom header presence without checking the client's protocol version, it could reject requests from older clients with a `400 Bad Request`. Server authors should ensure validation accounts for protocol version negotiation.

### Phase 3: Infrastructure Updates
Once both clients and internal servers support the headers, DevOps teams can update API Gateways and Load Balancers. Routing rules can now be based entirely on `Mcp-Name` and `Mcp-Param-*` headers, allowing enterprises to turn off deep packet inspection and dramatically improve network throughput.
