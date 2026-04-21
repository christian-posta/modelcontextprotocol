---
name: verify-summary
description: Fact-check a summary document against its source SEP, spec, or PR — verify all claims, normative levels, field names, code examples, and report errors
user_invocable: true
arguments:
  - name: file
    description: Path to the summary document to verify (e.g., updates-q2-2026/SEP-2575-Stateless-MCP.md)
    required: true
---

# Verifying a Summary Document

This skill fact-checks a summary/editorial document against its authoritative source material (SEP files, spec documents, PRs, issues). It catches fabricated details, stale names, wrong normative levels, incorrect code examples, and unsubstantiated claims.

**Do not skip steps.** The value of this skill is exhaustive cross-referencing — a cursory read is not sufficient.

## Phase 1 — Identify Sources

From the summary document, determine what it is summarizing:

1. **Extract the SEP number** from the filename or content (e.g., `SEP-2575`).
2. **Find the authoritative SEP file** in the repo:
   ```bash
   ls seps/*{number}* 2>/dev/null
   ```
3. **Find the PR** — SEPs are submitted as PRs. The PR diff is the most current version:
   ```bash
   gh pr list --search "SEP-{number}" --state all --limit 5
   gh pr diff {pr_number} 2>/dev/null | head -1000
   ```
   If the diff is large, fetch the full SEP file content from the PR.
4. **Find the related issue** (if any):
   ```bash
   gh issue list --search "SEP-{number}" --state all --limit 5
   ```
5. **Check if changes merged directly into spec docs** — some PRs modify `docs/specification/draft/` rather than adding a `seps/` file:
   ```bash
   gh pr view {pr_number} --json files
   ```
6. **Determine precedence:** PR file > issue body > spec docs. If the PR and issue diverge (e.g., a method was renamed in the PR), the PR is authoritative.

Record the PR status (OPEN/MERGED/CLOSED) and labels — these affect how the summary should frame the proposal.

## Phase 2 — Build the Checklist

Read the summary document and extract every verifiable claim into categories:

### A. Names and Identifiers
- RPC method names (e.g., `server/discover`, `subscriptions/listen`)
- TypeScript interface/type names (e.g., `DiscoverRequest`, `UnsupportedProtocolVersionError`)
- Field names, especially namespaced ones (e.g., `io.modelcontextprotocol/protocolVersion`)
- Error codes (e.g., `-32601`, `INVALID_PARAMS`)
- Notification names (e.g., `notifications/subscriptions/acknowledged`)

### B. Normative Levels
- Every MUST, SHOULD, MAY, MUST NOT, SHOULD NOT claim — verify the exact level against the source
- Watch for upgrades (summary says MUST, source says SHOULD) and downgrades

### C. Code Examples
- JSON-RPC examples must include `jsonrpc` and `id` fields
- HTTP examples: verify the POST path (Streamable HTTP uses a single endpoint, not per-method paths)
- Verify protocol version strings match what the SEP uses
- Verify `_meta` field names use correct namespace prefixes
- Verify request/response schemas match the SEP's TypeScript definitions
- Check that required fields are not omitted

### D. Architectural Claims
- "X replaces Y" — verify Y existed and X is the actual replacement
- "X is removed" — verify the SEP actually removes it
- Framing claims (e.g., "optional" vs "mandatory") — verify against source

### E. Editorial Claims
- Claims about motivation, rationale, or benefits — verify they match the SEP's stated motivation
- Claims about caching, performance, or behavior not explicitly in the SEP
- References to other SEPs — verify the relationship is stated in the source

## Phase 3 — Cross-Reference

For each item in the checklist, find the corresponding passage in the source material. Use exact quotes or line references.

**Common error patterns to watch for:**

1. **Stale names from issue vs PR:** The issue body often has an earlier draft. Method names, type names, and error codes frequently change between issue and PR. Always prefer the PR.
2. **Fabricated parameter structures:** Summaries often invent simplified parameter shapes (e.g., a `"topics"` array) when the actual SEP has a more complex structure.
3. **Wrong HTTP paths:** Streamable HTTP uses a single endpoint (typically `/mcp`), not per-method paths like `/mcp/tools/call`.
4. **Missing namespace prefixes:** `_meta` fields in newer SEPs use `io.modelcontextprotocol/` prefixes that summaries often drop.
5. **Fabricated protocol versions:** Summaries may use a version string that doesn't appear in the SEP.
6. **Missing required fields in examples:** The SEP may require fields (like `clientInfo`) that the summary's examples omit.
7. **Missing lifecycle steps:** E.g., an acknowledgment notification that must precede data flow.
8. **Unsubstantiated claims:** Claims about caching, performance characteristics, or behavior not stated in the SEP.

## Phase 4 — Report

Produce a structured report with two sections:

### Errors Found

For each error, report:
- **What the summary says** (quote the specific text)
- **What the source says** (quote or cite the specific passage)
- **Classification:** Wrong name / Wrong normative level / Fabricated detail / Incomplete example / Unsubstantiated claim / Stale (from issue, not PR)
- **Severity:** High (factually wrong, would mislead a reader) / Medium (imprecise but directionally correct) / Low (omission, not error)

### Verified Claims

List the claims that were checked and found accurate. This is important for the checkpoint file — it proves the review was thorough, not just error-hunting.

## Phase 5 — Fix

After presenting the report, ask the user if they want fixes applied. If yes:

1. Apply all edits to the summary document.
2. Update the checkpoint file (if one exists, e.g., `.claude/updates-checkpoint.md`) with:
   - The file reviewed and source material used
   - All fixes applied (numbered, with before/after)
   - All verified claims (confirms no changes needed)
   - PR status and link

## Verification Standards

- **Never fabricate source quotes.** If you can't find the source passage, say so.
- **Always check both the PR diff and the issue body** — note which one you're citing.
- **Protocol version strings must exactly match** what appears in the source.
- **Namespace prefixes matter** — `clientCapabilities` and `io.modelcontextprotocol/clientCapabilities` are different field names.
- **JSON-RPC examples must be valid JSON-RPC** — `jsonrpc: "2.0"` and `id` are required on requests.
- **HTTP examples must use correct paths** for the transport being described.
