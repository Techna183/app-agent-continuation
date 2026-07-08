# app-agent-continuation

Spec work for **Delegated Work Sessions (DWS)** — a proposed MCP
extension (plus a companion host-integration profile) that makes
app → assistant work continuation a first-class concept:

> An app deep-links the user into an assistant (ChatGPT, Claude, …)
> with signed context; the assistant does the work through the app's
> MCP server; structured results flow back server-side via a reserved
> completion tool; and the assistant returns focus to the app —
> `redirect_uri` semantics for agentic work.

## Contents

- [`spec/delegated-work-sessions.md`](spec/delegated-work-sessions.md) —
  the draft SEP (Extensions Track): motivation, two conformance
  profiles, wire-level schemas, terminal-state model, security
  considerations, and the graceful-degradation path that works against
  today's hosts with plain `?q=` deep links.

## Status

Draft / pre-proposal. Next steps per the MCP SEP process:

1. Float the idea in the MCP contributor Discord / Agents Working Group
   to find a sponsor.
2. Build the reference implementation (§12 of the spec): initiator app,
   reference MCP server, simulated host.
3. Submit as a GitHub issue per the SEP guidelines.
