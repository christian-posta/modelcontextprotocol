# Updates Q2 2026: OpenTelemetry Trace Context (SEP-414)

## Overview

SEP-414 documents conventions for how OpenTelemetry (OTel) distributed tracing context is propagated through MCP. When a complex AI system triggers a tool, it may traverse a Client, an AI Gateway, and a backend MCP Server. To track requests across these systems, OpenTelemetry uses standard identifiers (like `traceparent`). SEP-414 formalizes the existing convention that these identifiers are carried inside the `_meta` object of the JSON-RPC request parameters. This convention was already in practice in the C#, Python SDKs and other implementations before the SEP was written.

## Key Changes Introduced by SEP-414

1. **Standardized Carrier:** OTel trace context is propagated via the `_meta` object using W3C Trace Context and W3C Baggage formats.
2. **DNS Prefixing Exception:** The keys `traceparent`, `tracestate`, and `baggage` are explicitly granted an exception to the standard MCP rule that requires DNS-prefixing for custom metadata keys. They must be used exactly as named without prefixes.

## Why use `_meta` instead of HTTP Headers?

A common question is why MCP propagates trace context in the JSON-RPC `_meta` object rather than relying on standard HTTP headers (like `traceparent: ...`).

The SEP's stated rationale is **interoperability**: documenting the shared convention prevents divergent implementations (e.g., one SDK using `io.modelcontextprotocol.traceparent` while another uses bare `traceparent`), which would break traces and log correlation. The convention also aligns with the [OTel semantic conventions for MCP](https://opentelemetry.io/docs/specs/semconv/gen-ai/mcp/).

Additionally, using `_meta` provides **transport agnosticism** — MCP works over `stdio` where HTTP headers don't exist. Embedding trace identifiers in the JSON-RPC payload ensures telemetry context survives regardless of transport.

Note that the two approaches are not mutually exclusive: the related [SEP-2028](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2028) (still in progress) builds on SEP-414 to also forward `_meta` trace values into HTTP headers for intermediaries that expect them there.

## Concrete Examples of Trace Context

According to the specification, the `_meta` object is injected directly into the `params` object of a JSON-RPC Request or Notification. Here is what that looks like in practice.

### Example 1: A Tool Execution Request
When a client asks the server to execute a tool, it embeds the W3C `traceparent` (and optionally `tracestate` and `baggage`) alongside the tool's standard arguments.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "deploy_server",
    "arguments": {
      "region": "us-east-1"
    },
    "_meta": {
      "traceparent": "00-0af7651916cd43dd8448eb211c80319c-00f067aa0ba902b7-01",
      "tracestate": "rojo=00f067aa0ba902b7",
      "baggage": "userId=alice123,serverNode=DF28"
    }
  }
}
```

### Example 2: A Resource Read Request
Similarly, when requesting a resource from a backend system, the trace context ensures the backend database query can be correlated back to the AI's prompt generation.

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "resources/read",
  "params": {
    "uri": "file:///logs/system.log",
    "_meta": {
      "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
    }
  }
}
```

## Operational Considerations

While this SEP primarily documents existing conventions, there are practical considerations for implementers:

1. **DNS Prefixing Exception:** MCP's `_meta` key naming convention requires a DNS-style prefix (e.g., `com.example/my_key`) for custom keys to prevent collisions. The spec explicitly grants an exception for `traceparent`, `tracestate`, and `baggage` to maintain compatibility with W3C and OTel conventions. Implementations following the current draft spec will handle this correctly — the exception is built into the key format rules.
2. **HTTP Header Forwarding:** When intermediaries (like AI gateways) need to forward trace context to upstream HTTP services that expect standard W3C headers, they must extract the `_meta` values and map them to HTTP headers. This translation concern is being addressed separately by [SEP-2028](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2028), which is still in progress. Implementations like [Envoy AI Gateway](https://github.com/envoyproxy/ai-gateway) already handle this pattern.
3. **Baggage and Trust Boundaries:** The SEP notes that "trace context in `_meta` may include correlation IDs" and that "implementations should follow existing data-handling guidance appropriate to their environment." In practice, the OTel `baggage` property can carry arbitrary contextual data, so implementations connecting to untrusted third-party servers should consider whether forwarding `baggage` is appropriate for their trust model.

## Existing Implementations

This convention is already in production across the ecosystem. Reference implementations listed in the SEP include:

- [C# SDK instrumentation](https://github.com/modelcontextprotocol/csharp-sdk/blob/main/src/ModelContextProtocol.Core/Diagnostics.cs)
- [Python SDK instrumentation](https://github.com/modelcontextprotocol/python-sdk/pull/1693)
- [Envoy AI Gateway](https://github.com/envoyproxy/ai-gateway)
- [Logfire](https://github.com/pydantic/logfire) (Python observability)
- [OpenInference MCP instrumentation](https://github.com/Arize-ai/openinference) (Python and TypeScript)
