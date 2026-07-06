# Draft Q2 2026: Feature Lifecycle and Deprecation Policy (SEP-2596)

## Overview

SEP-2596 gives MCP its first formal, written policy for how a feature moves through its life: **Active → Deprecated → Removed**, with a guaranteed minimum support window and a public registry of what's on the way out. It's a governance/process SEP rather than a wire-protocol change, but it's the mechanism that makes every *other* deprecation in this draft (Roots/Sampling/Logging via SEP-2577, the HTTP+SSE transport, `includeContext` values, Dynamic Client Registration) safe and predictable instead of ad hoc.

Status: **Final**. (Also previously on your "pending candidates" list — it landed.)

## The Core Problem

Before this, "deprecated" in MCP had no defined meaning. There was no promise about how long a deprecated feature would keep working, no single place to see what was deprecated, and no rule for when something could actually be deleted. That makes it risky for implementers to depend on anything, and risky for maintainers to remove anything.

## Key Changes Introduced by SEP-2596

### 1. Three explicit feature states

- **Active** — normal, supported, safe to adopt.
- **Deprecated** — still fully functional and still in the spec, but new implementations should not adopt it; it's on a removal clock.
- **Removed** — deleted from the spec.

### 2. A minimum 12-month deprecation window

A feature MUST remain in the spec (as Deprecated) for at least twelve months / across the relevant version releases before it becomes eligible for removal. This is the guarantee SEP-2577 leans on when it says Roots/Sampling/Logging "remain fully functional during the deprecation window."

### 3. A deprecated-features registry

A dedicated page (`/specification/draft/deprecated`) tracks every feature currently in the Deprecated state, so there's a single source of truth for "what should I stop building on."

### 4. Reclassifications applied in this draft

Using the new policy, this release moves several already-soft-deprecated things into the formal **Deprecated** state:

- The **HTTP+SSE transport** (deprecated in practice since protocol `2025-03-26`) → formally Deprecated; migrate to Streamable HTTP.
- The `includeContext` values **`"thisServer"`** and **`"allServers"`** (soft-deprecated since `2025-11-25`) → Deprecated; omit the field or use `"none"`.

## Concrete Example: how a deprecation now reads

Before, a JSDoc `@deprecated` tag was just a hint with no teeth. Now it maps to a defined lifecycle with a clock and a registry entry:

```jsonc
/**
 * @deprecated Deprecated 2026-07-28 (SEP-2577). Deprecated state per SEP-2596;
 * remains supported for at least 12 months. Tracked in the deprecated-features registry.
 * Migration: use direct LLM provider APIs.
 */
interface CreateMessageRequest { /* ... */ }
```

## Why this matters

A protocol that many vendors depend on needs a credible promise about stability *and* a credible path to shed complexity. This policy provides both: implementers get a guaranteed runway (nothing they depend on vanishes with less than a year's notice and a registry entry), and maintainers get a legitimate, non-breaking way to retire features. It's the enabling policy behind the "prune the surface area" theme running through this whole draft.

## Related deprecation in this release (not part of 2596, but same theme)

**Dynamic Client Registration (RFC 7591) is deprecated** as a client-registration mechanism (PR #2858) in favor of **Client ID Metadata Documents**. DCR remains available for backward compatibility with authorization servers that don't yet support CIMD. This is directly relevant to the auth notes (SEP-837 application types, SEP-2352 multi-AS) — the registration story is shifting away from DCR. See `DCR-Deprecation-CIMD.md`.
