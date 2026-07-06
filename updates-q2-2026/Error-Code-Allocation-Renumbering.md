# Draft Q2 2026: Error-Code Allocation Policy & Renumbering (no dedicated SEP)

## Overview

This draft defines a **partitioning policy** for the JSON-RPC server-error range and **renumbers** the error codes that were introduced during the draft accordingly. It also promotes `HeaderMismatchError` into the schema (it previously existed only in transport prose). There's no standalone SEP number for the allocation policy — it arrived alongside the MRTR/stateless work (changelog "Minor #12") — but it's a real normative change, and it directly corrects a value in the existing **SEP-2243 (HTTP Headers)** note.

## The Core Problem

JSON-RPC reserves `-32000` to `-32099` as an implementation-defined "server error" range. MCP had started minting protocol-level error codes inside it (`-32001`, `-32003`, `-32004`, `-32042`, `-32002` for resource-not-found), which both collides with SDKs' own ad-hoc codes and blurs the line between "the MCP spec defines this" and "some server made this up."

## Key Changes

### 1. The range is partitioned

- **`-32000` to `-32019`** — remains **implementation-defined**. Existing SDK usage is grandfathered here.
- **`-32020` to `-32099`** — **reserved for the MCP specification**.

### 2. Draft-introduced codes are renumbered into the reserved band

| Error | Old code | New code |
|-------|----------|----------|
| `HeaderMismatch` (`HeaderMismatchError`) | `-32001` | **`-32020`** |
| `MissingRequiredClientCapability` (`MissingRequiredClientCapabilityError`) | `-32003` | **`-32021`** |
| `UnsupportedProtocolVersion` (`UnsupportedProtocolVersionError`) | `-32004` | **`-32022`** |

`HeaderMismatchError` is now a real schema type (was prose-only). Separately, resource-not-found moved `-32002` → `-32602` (SEP-2164), and the old `-32042` URL-elicitation-required code was removed (its notification was dropped under MRTR).

### 3. New error `data` shapes

- `MissingRequiredClientCapabilityError.data.requiredCapabilities: ClientCapabilities`
- `UnsupportedProtocolVersionError.data: { supported: string[], requested: string }`

## Correction to the existing SEP-2243 note

Your `SEP-2243-HTTP-Headers.md` shows the header-mismatch rejection using:

```jsonc
{ "error": { "code": -32001, "message": "HeaderMismatch: ..." } }
```

Under this draft that code is now **`-32020`**:

```jsonc
{ "error": { "code": -32020, "message": "HeaderMismatch: HTTP headers do not match request body parameters" } }
```

(These codes also map to an HTTP `400` at the transport layer.)

## Why this matters

It draws a clean, enforceable boundary between spec-owned and implementation-owned error codes, so clients can rely on the meaning of a `-3202x` code while existing SDK error codes in `-3200x` keep working untouched. It's a small but load-bearing bit of hygiene for a protocol with many independent implementations.
