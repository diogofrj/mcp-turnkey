# ADR 0008 — Tools that work in Code Mode clients

- Status: accepted
- Date: inherited from the template

## Context

Some MCP clients do not put tool schemas in the prompt. In Code Mode (Goose, Cloudflare
Agents) the model discovers tools on demand and writes a short script that calls them,
loops and filters in a sandbox, and returns only the final answer. The script consumes
the tool result as data, so free text or images that a person would read are hard or
impossible for it to use. Goose's Code Mode ignores non-text content entirely.

## Decision

1. **Results are typed JSON.** Every tool returns a dict or a list of dicts built from
   Pydantic models. FastMCP sends it both as `structuredContent` and as a JSON text
   block, so classic clients and Code Mode clients read the same payload.
2. **Errors are model-safe and immediate.** Domain failures become a `ToolError` with a
   short message; no tool waits on a human prompt or returns a partial human-readable
   dump.
3. **No image-only or binary-only results.** When a tool produces media, it returns a
   JSON object with a URL or id next to any media block.
4. **The standard MCP surface stays the only one.** This server does not implement
   meta-tools or run code; discovery and orchestration belong to the client.

## Consequences

- One tool contract serves classic and Code Mode clients.
- New tools follow the existing `_dump` / `ToolError` pattern in `server.py`; a tool that
  returns a plain string needs a reason in its review.
- Token savings depend on the client, not on this server.
