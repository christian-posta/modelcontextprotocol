# Draft Q2 2026: TTL for List Results (SEP-2549)

## Overview

SEP-2549 adds a transport-agnostic **freshness hint** to the results of list-style methods, so clients can cache them for a stated period instead of relying solely on push notifications. It introduces a new `CacheableResult` interface with two required fields — `ttlMs` and `cacheScope` — carried on `tools/list`, `prompts/list`, `resources/list`, `resources/read`, `resources/templates/list`, and `server/discover`.

This is the caching half of the sessionless story. Once SEP-2567 removed sessions, `tools/list` results no longer vary per connection, which makes them genuinely cacheable — but clients still needed a signal for *how long*. TTL is that signal.

Status: **Final**, Standards Track. Author: Caitie McCaffrey. (In your earlier review this was filed under "pending candidates" — it has since landed.)

## The Core Problem

The only way to know a list had changed used to be an SSE `notifications/*/list_changed` push. That has three weaknesses:

1. It's hard to maintain over HTTP transports (you need a live stream open just to hear about changes).
2. It adds implementation complexity on both ends.
3. It gives **no freshness signal** — a client that just connected has no idea whether the list it fetched is good for one second or one day, so it either over-fetches or risks staleness.

## Key Changes Introduced by SEP-2549

### 1. New `CacheableResult` interface (extends `Result`)

- **`ttlMs: number`** (minimum 0) — how many milliseconds the client MAY cache the response before re-fetching. Semantics mirror HTTP `Cache-Control: max-age`. `0` = immediately stale.
- **`cacheScope: "public" | "private"`** — who may cache. Mirrors HTTP `Cache-Control: public`/`private`. Absent → treated as `"public"`.

Both fields are **required** on the affected results (there's no safe implicit default for user-specific data, hence `cacheScope` is mandatory).

### 2. Which results become cacheable

`tools/list`, `prompts/list`, `resources/list`, `resources/read`, `resources/templates/list`, and `server/discover` now extend `CacheableResult`.

### 3. It complements — does not replace — `listChanged`

A received `notifications/*/list_changed` invalidates the cache regardless of remaining TTL. Servers that change a list before its TTL expires and that advertise `listChanged` SHOULD still send the notification.

## Concrete Example: `tools/list` with a 5-minute TTL

### Before

```jsonc
{
  "resultType": "complete",
  "tools": [ { "name": "get_weather", "inputSchema": { "type": "object", "properties": { /* ... */ } } } ],
  "nextCursor": "next-page-cursor"
}
```

The client had no idea how long this was valid; it either re-fetched constantly or kept an SSE stream open to hear about changes.

### After

```jsonc
{
  "resultType": "complete",
  "tools": [ { "name": "get_weather", "inputSchema": { "type": "object", "properties": { /* ... */ } } } ],
  "nextCursor": "next-page-cursor",
  "ttlMs": 300000,
  "cacheScope": "public"
}
```

The client may safely cache this for 5 minutes and share it across users (`public`). A tenant-specific list would instead set `"cacheScope": "private"` so shared intermediaries never serve it to a different user.

## Why this matters

TTL turns list discovery into a normal cacheable HTTP-style interaction, which is exactly what the sessionless architecture needs. It lets a fleet of subagents behind a gateway cache one server's tool list and reuse it — killing the `O(subagents × servers)` re-fetch storm that SEP-2567 called out — without requiring anyone to hold an SSE stream open just for change detection.

## Challenges / Considerations

1. **Cache-scope correctness is a security property.** A shared/proxy cache MUST NOT serve a `"private"` response to a different user. Getting `cacheScope` wrong leaks one tenant's tool/resource list to another.
2. **Consistency across pages.** A server MUST apply the same `cacheScope` to every page of a paginated result — you can't mark page 1 public and page 2 private.
3. **Don't turn TTL into a polling timer.** Clients SHOULD re-fetch lazily when stale on next access, and if they *do* poll they SHOULD add jitter/backoff; treating `ttlMs` as a fixed background poll interval would synchronize a fleet into a thundering herd.
4. **Backward compatible.** Older servers omit the fields → clients assume `ttlMs = 0` (always re-validate). No capability negotiation involved.
