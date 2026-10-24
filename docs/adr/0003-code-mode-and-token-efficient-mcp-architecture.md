# ADR 0003 — Token efficiency is a client concern; the template stays Code-Mode friendly

- Status: accepted
- Date: 2026-10-24

## Context

Clients that inject every tool schema into the prompt pay for the whole catalog on every
turn. Several clients now avoid that: Claude Code loads MCP tool schemas on demand, and
Goose's Code Mode replaces the catalog with three meta-tools (`list_functions`,
`get_function_details`, `execute_javascript`) and runs model-written JavaScript that calls
the real tools in a Deno sandbox.

Measured on 2026-10-24 (Goose 1.56.0, `cc/claude-haiku-5-5` through OmniRoute, 23 MCP
servers / 584 tools, read-only tasks, 2 runs each; prompt tokens = input + cache write):

| Task | Mode | Prompt tokens | Cost (USD) | Wall (s) | Correct |
|---|---|---:|---:|---:|---|
| Reply "OK", no tools | Traditional | 64,752 | 0.0081 | 11.2 | 2/2 |
| | Code Mode | 28,315 | 0.0019 | 7.1 | 2/2 |
| List 40 NPM proxy hosts, count and filter | Traditional | 162,506 | 0.0095 | 22.5 | **0/2** |
| | Code Mode | 228,670 | 0.0108 | 17.1 | 2/2 |
| Search memory, filter by kind | Traditional | 278,546 | 0.0125 | 29.7 | 2/2 |
| | Code Mode | 229,948 | 0.0105 | 13.8 | 2/2 |

Findings:

1. The fixed overhead drops by about 56%, not 99%: the meta-tools, the system prompt and
   the `list_functions` output (every function name in the catalog) still cost tokens.
2. Multi-step tasks save about 17% of tokens and about 50% of wall time, mostly because
   filtering happens in the sandbox. Not a step change.
3. The largest effect is reliability: with 584 tools in the prompt the model failed to
   find tools it had (it tried to "enable" an already loaded extension, or called a
   non-prefixed tool name). With Code Mode it found them every time.

## Decision

1. **The template keeps the standard MCP surface** (`tools/list`, `tools/call`). It does
   not ship a meta-tool proxy or a code sandbox: running model-written code is the
   client's job, and every client that does it (Goose, Claude Code, Cloudflare Agents)
   brings its own runtime.
2. **The template makes its tools easy to drive from code**, see the template's ADR 0008:
   typed JSON results, model-safe errors, short descriptions.
3. **Catalog size is managed where the catalog lives**: enable only the servers a harness
   needs, and prefer clients with on-demand discovery or Code Mode when a harness carries
   many servers.

## Consequences

- No new runtime, dependency or attack surface in the template.
- The numbers above are a single small sample (one model, n=2). Re-measure before using
  them as a target; the method is the same three tasks, with and without
  `--with-builtin code_execution`.
- Goose Code Mode ignores non-text tool content, so tools that only return images or
  binaries are invisible to it.
