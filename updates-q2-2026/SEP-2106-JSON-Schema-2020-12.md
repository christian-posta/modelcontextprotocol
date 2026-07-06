# Draft Q2 2026: JSON Schema 2020-12 for Tool Schemas (SEP-2106)

## Overview

SEP-2106 loosens the constraints MCP places on a tool's `inputSchema` and `outputSchema` so they can use the **full JSON Schema 2020-12 dialect** — composition (`oneOf`/`anyOf`/`allOf`/`not`), conditionals (`if`/`then`/`else`), and references (`$ref`/`$defs`/`$anchor`) — and it relaxes `structuredContent` to allow non-object values.

Status: **Final**, Standards Track. Author: John McBride (original), Ola Hungerford (shepherd).

## The Core Problem

Previously MCP defined a tool's schemas as an intentionally tiny subset of JSON Schema: essentially `{ type: "object", properties, required }`. That had two painful consequences:

1. **No composition.** You couldn't express "exactly one of `id` or `name`" (`oneOf`), or reuse a shared sub-schema via `$ref`. SDKs like FastMCP grew error-prone workarounds to smuggle richer schemas through.
2. **Objects only, everywhere.** A tool that logically returns *a list* had to wrap it in a container object (`{ "results": [...] }`) purely to satisfy the "must be an object" rule, and `structuredContent` was locked to a JSON object so array/scalar payloads couldn't be represented at all.

## Key Changes Introduced by SEP-2106

### 1. `inputSchema` allows any 2020-12 keywords (root still an object)

Was `{ $schema?, type: "object", properties?, required? }`. Now `{ $schema?, type: "object", [key: string]: unknown }` — the root MUST still be `type: "object"`, but any 2020-12 keyword is now permitted alongside it.

### 2. `outputSchema` no longer requires `type: "object"`

Was `{ $schema?, type: "object", properties?, required? }`. Now `{ $schema?, [key: string]: unknown }` — the root object requirement is **dropped**, so a tool can declare it returns an array or a primitive.

### 3. `structuredContent` widened to `unknown`

Was `{ [key: string]: unknown }` (object only). Now **`unknown`** — objects, arrays, or scalars are all valid. This applies to both `CallToolResult.structuredContent` and `ToolResultContent.structuredContent`.

### 4. Guardrails for the new expressiveness

- Implementations MUST NOT automatically dereference a `$ref` that resolves to a **network URI** (SSRF / fetch-DoS). Opt-in network deref MAY be offered but MUST be off by default and SHOULD use an allowlist + timeouts + size limits.
- Implementations SHOULD bound composition-keyword validation (max depth, subschema cap, time budget) to prevent CPU exhaustion.
- Implementations MUST still validate inputs/outputs against the declared schemas.

## Concrete Example: things you couldn't express before

### `oneOf` in an input schema (find by id OR by name)

```json
{
  "name": "find_resource",
  "title": "Resource Finder",
  "description": "Find a resource by ID or name",
  "inputSchema": {
    "type": "object",
    "oneOf": [
      { "properties": { "id":   { "type": "string" } }, "required": ["id"] },
      { "properties": { "name": { "type": "string" } }, "required": ["name"] }
    ]
  }
}
```

### An array-typed output schema (no more container-object wrapping)

```json
{
  "name": "list_users",
  "title": "User List",
  "inputSchema": { "type": "object", "properties": {} },
  "outputSchema": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "id":    { "type": "string" },
        "name":  { "type": "string" },
        "email": { "type": "string" }
      },
      "required": ["id", "name", "email"]
    }
  }
}
```

The corresponding result can now put the array directly in `structuredContent` (previously impossible):

```jsonc
{
  "resultType": "complete",
  "structuredContent": [ { "id": "u1", "name": "Ada", "email": "ada@x.com" } ],
  "content": [ { "type": "text", "text": "[{\"id\":\"u1\",\"name\":\"Ada\",\"email\":\"ada@x.com\"}]" } ]
}
```

## Why this matters

Tool authors get real JSON Schema instead of a crippled subset — mutually-exclusive argument sets, shared definitions, and list/primitive return values all become first-class. It also aligns MCP with what LLM providers' function-calling schemas already accept, so SDKs can stop maintaining translation shims.

## Challenges for Existing Servers

1. **Source-breaking for typed TypeScript consumers.** Widening `structuredContent` from an object to `unknown` means code that did `result.structuredContent.someField` no longer type-checks — you must narrow first. Wire format is unchanged, but typed clients need updates.
2. **Old-client fallback is mandatory.** A server returning array/primitive `structuredContent` MUST also emit a `TextContent` block containing the serialized JSON, so clients that predate this change still get the data.
3. **Validation cost and SSRF.** Supporting arbitrary composition and `$ref` means servers/clients must add the depth/time bounds and the no-network-deref default, or they inherit new DoS/SSRF exposure.
