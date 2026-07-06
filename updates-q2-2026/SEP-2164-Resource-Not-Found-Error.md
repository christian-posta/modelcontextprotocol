# Draft Q2 2026: Standardize the Resource-Not-Found Error Code (SEP-2164)

## Overview

SEP-2164 changes the recommended JSON-RPC error code for "resource does not exist" from **`-32002`** to **`-32602` (Invalid Params)**, aligning MCP with the JSON-RPC specification's meaning for that range.

Status: **Final**, Standards Track. Author: Peter Alexander.

## The Core Problem

The spec previously recommended `-32002` for a missing resource on `resources/read`. But `-32000` to `-32099` is JSON-RPC's *implementation-defined server-error* range — it's not supposed to carry protocol-defined semantics. The result was inconsistency across SDKs: only about half used `-32002` at all; TypeScript already returned `-32602`, Python returned `0`, Kotlin returned `-32603`. A client couldn't reliably detect "resource not found" from the code.

## Key Changes Introduced by SEP-2164

1. Servers **MUST** return **`-32602`** (Invalid Params) when a requested resource doesn't exist (replacing the `-32002` recommendation).
2. The error's `data` field **SHOULD** include the `uri` that was not found.
3. Servers **MUST NOT** return an empty `contents` array to signal not-found — that's ambiguous and is no longer acceptable.
4. During the transition, clients **SHOULD** accept both `-32602` and the legacy `-32002`.

## Concrete Example

### Before

```jsonc
{ "jsonrpc": "2.0", "id": 1,
  "error": { "code": -32002, "message": "Resource not found" } }
```

...or, in some SDKs, an ambiguous empty result:

```jsonc
{ "jsonrpc": "2.0", "id": 1, "result": { "contents": [] } }   // was this "empty" or "missing"?
```

### After

```jsonc
{ "jsonrpc": "2.0", "id": 1,
  "error": {
    "code": -32602,
    "message": "Resource not found",
    "data": { "uri": "file:///does/not/exist.txt" }
  } }
```

## Why this matters

It makes a common failure mode reliably machine-detectable and stops the empty-array ambiguity, without inventing a new code — `-32602` (Invalid Params) is the natural JSON-RPC fit since the requested `uri` is, in effect, an invalid parameter. Note this is part of the broader error-code cleanup in this draft (see `Error-Code-Allocation-Renumbering.md`), which reserves `-32020`..`-32099` for MCP and leaves `-32000`..`-32019` implementation-defined.

## Considerations

- Practical impact is small precisely *because* `-32002` was never followed consistently — but clients must keep accepting `-32002` until servers have all moved.
- No security implications.
