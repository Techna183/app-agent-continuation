# Sidequest — The App–Agent Continuation Protocol

**Your app is the main quest.**

Sidequest is a proposed open protocol (an MCP extension plus a
companion host-integration profile) that lets a native or web app
dispatch a bounded unit of agentic work — a *sidequest* — to the
user's AI agent of choice, and get both the results and the user back:

> The app deep-links the user into their agent (ChatGPT, Claude, …)
> with signed context. The agent does the work through the app's MCP
> server. Structured results flow back server-side via a reserved
> completion tool — before the user even taps "return." The agent then
> hands focus back to the app. `redirect_uri` semantics for agentic
> work.

## Why

App developers are being offered a false dichotomy:

1. **Move everything into the agent surface** — and give up the rich
   native experience (direct manipulation, visualization, latency,
   platform integration, your own brand) that apps are actually good at.
2. **Build an agent into your app** — a massive lift (inference costs,
   agentic-loop engineering, streaming chat UX, memory, safety) whose
   market outcome is a thousand mediocre in-app chatbots, while users
   already have an agent they chose, trust, and pay for.

Sidequest is the third path: stay native when native wins, and when a
moment genuinely calls for an agentic chat experience, hand it off to
the user's agent of choice — scoped tools, signed context, structured
results, focus returned. Best of both worlds, no rebuilding either one.

## Contents

- [`spec/sidequest-protocol.md`](spec/sidequest-protocol.md) —
  the draft SEP (Extensions Track): strategic motivation, two
  conformance profiles, wire-level schemas, terminal-state model,
  security considerations, and the graceful-degradation path that
  works against today's hosts with plain `?q=` deep links.

## Status

Draft / pre-proposal. Next steps per the MCP SEP process:

1. Float the idea in the MCP contributor Discord / Agents Working Group
   to find a sponsor.
2. Build the reference implementation (§12 of the spec): initiator app,
   reference MCP server, simulated host.
3. Submit as a GitHub issue per the SEP guidelines.
