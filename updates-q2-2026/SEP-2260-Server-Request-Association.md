# Updates Q2 2026: Server Request Association (SEP-2260)

## Overview

SEP-2260 introduces a strict constraint on how and when an MCP Server can initiate requests back to the Client. Specifically, it dictates that server-initiated requests like `roots/list`, `sampling/createMessage`, and `elicitation/create` **MUST** be nested within an active, originating client-to-server request. 

This SEP officially prohibits "unsolicited" or "standalone" server requests. A server cannot wake up in the background and spontaneously ask the client's LLM to generate a message, nor can it spontaneously ask the client for a list of its root directories.

*Note: The operational `ping` request is explicitly exempted from this rule and can be sent at any time.*

## Key Changes Introduced by SEP-2260

1. **The "MUST" Requirement:** Previous versions of the specification stated that server messages *SHOULD* relate to an originating client request. This is now a strict **MUST**. 
2. **Prohibition of Standalone Streams:** For Streamable HTTP transports, the GET-initiated standalone SSE streams can no longer be used to send `roots/list`, `sampling/createMessage`, or `elicitation/create`.
3. **Client Rejection:** Clients receiving unsolicited server-to-client requests (requests not bound to an active client operation) **SHOULD** reject them with a `-32602 (Invalid Params)` JSON-RPC error.

## Concrete Examples

To understand the practical impact, here is how the "nested" rule applies to both Sampling and Elicitation.

### Example 1: LLM Sampling
**✅ ALLOWED (Nested Pattern):** A user asks the AI to summarize a file. The AI client calls the `analyze_data` tool on the MCP Server. While processing that specific tool call, the server requests LLM assistance.
```python
@mcp.tool()
async def analyze_data(data: str, ctx: Context) -> str:
    # Valid: Requesting LLM assistance while processing an active tool call
    result = await ctx.session.create_message(
        messages=[SamplingMessage(role="user", content="Format this data: ...")]
    )
    return result.content.text
```

**❌ PROHIBITED (Standalone Pattern):** A server has a background cron job running that attempts to sample a message every 60 seconds without any active client prompt.
```python
async def background_task():
    while True:
        await asyncio.sleep(60)
        # Invalid: Initiating sampling without a client request context
        await session.create_message(...)
```

### Example 2: User Elicitation (The Lazy Pattern)
Elicitation (`elicitation/create`) allows a server to pause and ask the human user for input (like a missing token, a URL, or a configuration choice). Previously, some servers would trigger this immediately upon connection startup. Under SEP-2260, they must use a "lazy evaluation" pattern.

**✅ ALLOWED (Lazy Elicitation):** The server waits until the user tries to use a feature that requires the information, and *then* suspends the tool execution to ask for it.
```python
@mcp.tool()
async def deploy_app(ctx: Context) -> str:
    if not has_deployment_url():
        # Valid: Eliciting human input because the user actively requested a deployment tool
        response = await ctx.session.elicitation_create(
            prompt="Please provide the target deployment URL to proceed:"
        )
        save_deployment_url(response.text)
        
    return do_deploy()
```

**❌ PROHIBITED (Startup Elicitation):** The server asks for user input immediately upon connection initialization, before the client has actually called any tools or requested any resources.
```python
async def on_initialize(session: Session):
    # Invalid: Asking for user input out-of-band before the client initiates a request
    user_url = await session.elicitation_create(
        prompt="Welcome to the server! What is your deployment URL?"
    )
```

## Why this matters

Formalizing this "nested-only" requirement brings three massive benefits to the MCP ecosystem:

1. **Security & User Trust:** It eliminates a major security surface area. If a server could spontaneously request `sampling/createMessage`, a rogue server could spam the user's LLM (costing them money) or attempt prompt injection attacks while the user is away from their keyboard. Users now know that sampling/elicitation only happens *because* they initiated an action.
2. **Transport Simplification:** It drastically simplifies how network transports (like HTTP or WebSockets) have to be built. Transports no longer need to maintain complex, persistent bidirectional connections just in case a server wants to push a request. They only need to handle request-scoped communication.
3. **Lazy Elicitation Pattern:** For implementers that previously performed URL Elicitation immediately after server startup, SEP-2260 requires that they lazily defer these requests until the client actually calls a tool or resource that requires that elicited information.

## Timeout Considerations for Infrastructure

Because server requests are now strictly nested *inside* client requests, a single client request (like `tools/call`) might now block for an extended period if the server triggers a Human-in-the-Loop elicitation or a slow sampling task. 

DevOps teams and Server Implementers must ensure that transport timeouts (like Load Balancer idle timeouts) are sufficiently long to accommodate unbounded human-in-the-loop delays. Servers should utilize transport-level SSE keepalive mechanisms or protocol-level `ping` requests to prevent load balancers from dropping the connection while waiting for the nested request to resolve.
