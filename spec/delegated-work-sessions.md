# SEP-XXXX: Delegated Work Sessions (DWS)

| Field | Value |
|---|---|
| **Title** | Delegated Work Sessions: App-to-Assistant Work Continuation |
| **Track** | Extensions |
| **Extension ID** | `io.github.techna183.dws` (placeholder; final reverse-DNS ID assigned at adoption) |
| **Status** | Draft / Pre-proposal (RFP) |
| **Author(s)** | techna183 |
| **Created** | 2026-07-08 |
| **Requires** | MCP base protocol (rev 2025-06-18 or later); extension capability negotiation |
| **Replaces** | — |

---

## 1. Abstract

This proposal defines **Delegated Work Sessions (DWS)**: a first-class,
interoperable pattern by which a native or web application (the
**Initiator**) hands a unit of work to an AI assistant host application
(the **Host** — e.g. ChatGPT, Claude, or any MCP-capable chat surface),
the assistant performs that work against the Initiator's backend via the
Initiator's MCP server, and — on completion — structured results flow
back to the Initiator and user focus is returned to the Initiating
application.

Conceptually, DWS specifies **`redirect_uri` semantics for agentic
work**: the analog of the OAuth 2.0 authorization code flow
([RFC 6749]) or Android's `startActivityForResult`, applied to a
delegated conversational work session instead of an authorization
grant.

The specification is split into two conformance profiles, reflecting a
deliberate boundary:

1. **Profile A — MCP Extension (normative for MCP participants):**
   session binding of tool calls via `_meta`, a completion contract
   (terminal result schema + lifecycle notifications), and timeout /
   abandonment semantics. These are things the MCP wire protocol can
   actually govern.
2. **Profile B — Host Integration Profile (companion; adopted
   voluntarily by Hosts):** a standard launch envelope (deep-link /
   URL format) for initiating a session with signed context, and host
   behavior for returning focus to the Initiator on completion. These
   are host *application* behaviors that sit outside the MCP
   client↔server wire format but are required for the end-to-end flow.

---

## 2. Motivation

### 2.1 The problem

Today, an application that wants an assistant to perform work on the
user's behalf *inside the assistant's own UI* has only ad-hoc pieces:

- **Handoff in** is limited to unspecified, lossy, host-specific query
  parameters (`https://chatgpt.com/?q=...`,
  `https://claude.ai/new?q=...`). These carry only free text, have no
  integrity protection, no session correlation, no capability request,
  and behave inconsistently — particularly in mobile apps, where the
  parameter is frequently dropped.
- **Data return** can be improvised: since the Host talks to the
  Initiator's MCP server, developers define an unofficial "terminal
  tool" (e.g. `finalize_session(session_id, results)`) and instruct
  the model to call it. This works but is pure convention — no schema,
  no distinction between success, partial completion, abandonment, or
  error, and no way for the Initiator's server to learn that a user
  silently closed the tab.
- **Focus return** has no primitive at all. The best available today
  is a widget button or a link the model emits, both host-specific.

The result: every app/assistant pair reinvents a fragile,
non-interoperable version of the same flow. There is no equivalent of
the well-understood OAuth round trip — *hand off → do work → return
with result* — for agentic sessions.

### 2.2 Prior art

| Prior art | What DWS borrows |
|---|---|
| **OAuth 2.0 authorization code flow** ([RFC 6749], [RFC 9700]) | Registered return URIs, `state` correlation, open-redirect defenses, replay protection |
| **Android `startActivityForResult` / `ActivityResult` API** | Typed request/result contract between two apps, cancellation as a first-class outcome |
| **OpenID Connect `request` object (JWS)** ([OIDC Core §6]) | Signed request payload so the receiving party can verify the Initiator |
| **MCP SEP-414 (trace context via `_meta`)** | Established pattern for threading cross-cutting correlation data through tool calls in `_meta` |
| **W3C Web Share Target / App Links / Universal Links** | Platform-verified association between a return URI and the Initiating app |

### 2.3 Why an Extensions-Track SEP

DWS is too opinionated for MCP core: it prescribes an
application-level workflow, not a transport or message primitive. The
Extensions Track exists precisely for interoperable patterns that
version independently, are negotiated through the client/server
`capabilities.extensions` map, and can graduate toward core if adoption
demonstrates demand. Agent workflows and the apps ecosystem are within
the project's stated priority areas; DWS is positioned as *interop for
agentic app-to-assistant workflows*.

### 2.4 Non-goals

- **Replacing direct API integration.** If an application wants a fully
  owned loop, calling a model API directly with the same tools remains
  the right architecture. DWS targets the case where the user should
  work *inside* the assistant's surface (its UI, subscription,
  memory, and multimodal capabilities).
- **Automatic, gesture-free focus return.** Browsers and mobile
  operating systems require a user gesture for most cross-app
  navigation. DWS specifies how the Host must *surface* the return
  affordance and when it MAY auto-redirect; it does not pretend the
  platform constraints away.
- **Assistant-to-assistant delegation.** Out of scope for v1.

---

## 3. Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL
NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and
**OPTIONAL** are to be interpreted as described in [RFC 2119] /
[RFC 8174].

| Term | Definition |
|---|---|
| **Initiator** | The application (native, web, or backend) that launches a delegated work session and expects a result. |
| **Host** | The assistant application (e.g. ChatGPT, Claude) in which the user performs the delegated work. The Host embeds the **MCP Client**. |
| **Server** | The Initiator-controlled MCP server the Host connects to during the session. The Initiator and Server are operated by the same party (or a trust relationship exists between them). |
| **Session** | A single delegated unit of work, identified by a `session_id`, from launch to a terminal state. |
| **Launch Envelope** | The signed, structured payload the Initiator passes to the Host at handoff (Profile B). |
| **Completion Record** | The structured terminal result delivered to the Server (Profile A). |
| **Return URI** | The pre-registered URI at which the Initiator regains focus (Profile B). |
| **Terminal state** | One of `completed`, `partial`, `error`, `abandoned`, `expired`, `declined`. |

---

## 4. Architecture Overview

```mermaid
sequenceDiagram
    autonumber
    participant App as Initiator App
    participant BE as Initiator Backend<br/>(incl. MCP Server)
    participant Host as Host (ChatGPT / Claude)
    participant Model as Model

    App->>BE: create session (context, requested tools, return_uri)
    BE-->>App: session_id + signed Launch Envelope
    App->>Host: launch via DWS deep link (Profile B §7)
    Host->>Host: validate envelope, confirm with user
    Host->>BE: MCP initialize (extensions: dws), session/attach
    loop delegated work
        Model->>BE: tools/call (…, _meta: dws session binding §6.2)
        BE-->>Model: structured results
    end
    Model->>BE: tools/call dws__complete_session (Completion Record §6.3)
    BE-->>App: webhook / push / poll: session terminal state
    Host->>App: focus return via Return URI (Profile B §8)
    Note over App: state already present when user lands
```

Key architectural observation (from which the two-profile split
follows): **the data path back to the Initiator never depends on the
Host exporting anything.** Because the Server is the Initiator's own
infrastructure and is in the loop for every tool call, the "result
callback" is simply a specified, reserved tool invocation plus
lifecycle notifications. Only the *launch* and the *focus return*
require Host cooperation.

---

## 5. Capability Negotiation (Profile A)

DWS is negotiated as an MCP extension. Servers declare support in
`initialize`:

```json
{
  "capabilities": {
    "extensions": {
      "io.github.techna183.dws": {
        "version": "1.0",
        "roles": ["server"],
        "lifecycleNotifications": true
      }
    }
  }
}
```

Clients (Hosts) that implement Profile A declare the same extension key
in their client capabilities.

- A Server that declares DWS **MUST** implement §6 in full.
- A Client that declares DWS **MUST** thread session binding metadata
  (§6.2) on every tool call within a session and **MUST** emit
  lifecycle notifications (§6.4) if `lifecycleNotifications` was
  negotiated.
- If the client does **not** declare DWS, the Server **MUST** still
  function (graceful degradation, §9): session binding falls back to
  prompt-carried tokens and lifecycle notifications are unavailable.

---

## 6. Profile A — MCP Extension (Normative)

### 6.1 Session establishment

A Session is created **by the Initiator's backend**, not by the Host.
The backend generates:

- `session_id` — an opaque identifier with at least 128 bits of
  entropy. It **MUST NOT** encode user data. It functions as a
  bearer correlation token and **MUST** be treated as a secret until
  the session reaches a terminal state.
- an expiry (`expires_at`) — sessions **MUST** have a bounded
  lifetime; **RECOMMENDED** default 30 minutes, maximum 24 hours.

The Server learns that a Host has picked up the session in one of two
ways:

1. **Extension-aware Host:** the client sends a
   `dws/session/attach` request after `initialize`:

   ```json
   {
     "jsonrpc": "2.0",
     "id": 7,
     "method": "extensions/io.github.techna183.dws/session/attach",
     "params": {
       "sessionId": "s_8f4b…",
       "envelopeSignature": "…"
     }
   }
   ```

   The Server **MUST** verify the `session_id` exists, is unexpired,
   and has not already been attached (single-use attach; see replay
   protection, §10.3). On success it responds with the session's
   working parameters:

   ```json
   {
     "result": {
       "sessionId": "s_8f4b…",
       "expiresAt": "2026-07-08T05:00:00Z",
       "allowedTools": ["stargraph_query", "stargraph_plan", "dws__complete_session"],
       "instructions": "Optional server guidance for the model, scoped to this session."
     }
   }
   ```

2. **Legacy Host (degradation path, §9):** the `session_id` arrives
   inside the prompt text and reaches the Server as an argument on the
   first tool call. The Server marks the session attached on first
   authenticated use.

### 6.2 Session binding of tool calls

Every tool call the client makes within a session **MUST** carry the
binding in `_meta`, following the established `_meta` propagation
pattern:

```json
{
  "method": "tools/call",
  "params": {
    "name": "stargraph_query",
    "arguments": { "target": "NGC 6543" },
    "_meta": {
      "io.github.techna183.dws/session": {
        "sessionId": "s_8f4b…",
        "seq": 4
      }
    }
  }
}
```

- `sessionId` (REQUIRED) — the attached session.
- `seq` (OPTIONAL) — monotonically increasing per-session sequence
  number, enabling the Server to detect gaps and ordering.

Servers **MUST** reject tool calls bound to expired or terminal
sessions with a JSON-RPC error (code `-32011`, "session not active"),
and **SHOULD** scope tool availability (via `tools/list` filtering) to
the session's `allowedTools`.

### 6.3 Completion contract

The Server **MUST** expose a reserved completion tool. Its name is
fixed so Hosts and models can recognize it generically:

**Tool: `dws__complete_session`**

```json
{
  "name": "dws__complete_session",
  "description": "Finalize the current delegated work session and deliver results to the initiating application. Call exactly once, when the delegated task is finished or cannot proceed.",
  "inputSchema": {
    "type": "object",
    "required": ["sessionId", "status"],
    "properties": {
      "sessionId": { "type": "string" },
      "status": {
        "type": "string",
        "enum": ["completed", "partial", "error", "declined"]
      },
      "summary": {
        "type": "string",
        "description": "Human-readable summary of what was done."
      },
      "result": {
        "type": "object",
        "description": "Structured results. Schema is Initiator-defined and advertised per session (resultSchema in the Launch Envelope)."
      },
      "error": {
        "type": "object",
        "properties": {
          "code": { "type": "string" },
          "message": { "type": "string" }
        }
      }
    }
  }
}
```

Semantics:

- `completed` — the delegated task finished as requested.
- `partial` — some work concluded; `result` contains what exists and
  `summary` explains the gap. Distinct from `error` so Initiators can
  resume rather than retry.
- `error` — the task failed; `error` is REQUIRED.
- `declined` — the model or user chose not to perform the task (e.g.
  policy refusal, user said "never mind"). No result expected.

The Server's response to `dws__complete_session` **SHOULD** include the
focus-return affordance so extension-aware Hosts can render it (§8),
and — for Hosts supporting embedded UI (e.g. Apps-SDK-style widgets) —
**MAY** include UI metadata rendering a "Return to *App*" control:

```json
{
  "content": [
    { "type": "text", "text": "Session finalized. The user can return to Stargraph." }
  ],
  "structuredContent": {
    "io.github.techna183.dws/return": {
      "returnUri": "https://app.stargraph.example/dws/return?session=s_8f4b…",
      "label": "Return to Stargraph"
    }
  }
}
```

Two abandonment-adjacent terminal states are **Server-assigned**, never
model-assigned:

- `expired` — `expires_at` passed without completion.
- `abandoned` — the client signaled session teardown (§6.4) or the
  Server observed prolonged inactivity (**RECOMMENDED** inactivity
  threshold: 15 minutes without a bound tool call) with no completion.

A session **MUST** reach exactly one terminal state, after which it is
immutable. `dws__complete_session` on an already-terminal session
**MUST** return error `-32012` ("session already terminal").

### 6.4 Lifecycle notifications (extension-aware Hosts)

Clients that negotiated `lifecycleNotifications` **MUST** send:

| Notification | When |
|---|---|
| `extensions/io.github.techna183.dws/session/closed` | The user closed the conversation/tab, or the Host is tearing down the connection with the session non-terminal. Params: `{ "sessionId", "reason": "user_closed" \| "host_shutdown" \| "user_cancelled" }` |

This fixes the silent-abandonment hole: today a Server cannot
distinguish "user is thinking" from "user closed the tab" except by
timeout. Best-effort delivery is acknowledged — Hosts crash — which is
why the inactivity timeout in §6.3 remains REQUIRED as backstop.

### 6.5 Result delivery to the Initiator

How the Initiator's backend relays the Completion Record to the
Initiator app (webhook, push notification, polling on `session_id`,
server-sent events) is **out of scope** — both ends are operated by the
same party. The spec's requirement is only:

- The Server **MUST** persist the Completion Record and terminal state
  at least until the Initiator acknowledges retrieval or the record
  ages out per the Initiator's policy.

This enables the signature UX of DWS: **the Initiator can resume work
before the user physically returns**, because data return (server-side)
precedes focus return (user gesture).

---

## 7. Profile B — Host Integration: Launch (Companion, Normative for Adopting Hosts)

### 7.1 Launch URL

Hosts adopting Profile B **MUST** accept a DWS launch at a documented
endpoint:

```
https://<host>/dws/launch?envelope=<base64url(JWS)>
```

and **SHOULD** also register a platform-native entry point (Android
App Link / iOS Universal Link to the same path) so native Initiators
get in-app handoff rather than a browser bounce. This addresses the
single most-reported failure of today's `?q=` mechanism: silent
parameter loss in mobile apps.

### 7.2 Launch Envelope

The envelope is a **JWS (compact serialization)** signed by the
Initiator. Payload claims:

```json
{
  "iss": "https://app.stargraph.example",
  "aud": "https://chatgpt.com",
  "iat": 1783822800,
  "exp": 1783824600,
  "jti": "s_8f4b…",

  "dws": {
    "version": "1.0",
    "sessionId": "s_8f4b…",
    "app": {
      "name": "Stargraph",
      "iconUri": "https://app.stargraph.example/icon.png"
    },
    "mcp": {
      "serverUrl": "https://mcp.stargraph.example/",
      "requestedTools": ["stargraph_query", "stargraph_plan"]
    },
    "prompt": "Plan tonight's observation session for the attached target list.",
    "context": {
      "contentType": "application/json",
      "data": { "targets": ["NGC 6543", "M27"], "site": "…" }
    },
    "resultSchema": { "$ref": "https://app.stargraph.example/schemas/plan-result.json" },
    "returnUri": "https://app.stargraph.example/dws/return"
  }
}
```

Requirements:

- `iss` **MUST** be an origin at which the Initiator publishes its
  signing keys via JWKS at
  `<iss>/.well-known/dws-initiator` (which also lists registered
  `returnUri` prefixes and MCP server URLs — the OAuth client-
  registration analog, but discovery-based rather than requiring a
  pre-existing Host↔Initiator relationship).
- Hosts **MUST** verify the signature against the `iss` JWKS, check
  `aud`, `exp`, and reject replayed `jti` values (§10.3).
- `context.data` is size-bounded: Hosts **MUST** accept ≤ 16 KiB and
  **MAY** reject larger. Larger context belongs behind the MCP server
  (fetched by tools), not in the URL.
- `prompt` and `context` are **attacker-influenceable text entering a
  model** and are subject to §10.5.

### 7.3 Host behavior at launch

On receiving a valid envelope, the Host:

1. **MUST** require an authenticated user (or complete login first,
   preserving the envelope).
2. **MUST** display a consent interstitial before starting the
   session, showing: the Initiator's verified origin (`iss`), app
   name, the MCP server it will connect to, the requested tools, and
   the return URI. This is the OAuth consent-screen analog and
   **MUST NOT** be skippable on first use of a given `iss`.
3. **MUST** connect to `mcp.serverUrl` (completing MCP authorization
   as normal — DWS does not alter MCP auth), negotiate the DWS
   extension, and send `session/attach` (§6.1).
4. **MUST** start the conversation with `prompt` + `context`
   presented to the model, clearly provenance-marked as
   third-party-app-provided content (not user-authored, not
   system-privileged).
5. If the user declines consent, the Host **SHOULD** notify the
   Server (`session/closed`, reason `user_cancelled`) if reachable
   without auth friction, else simply not attach — the session then
   expires (§6.3).

---

## 8. Profile B — Host Integration: Focus Return

1. When the Server's `dws__complete_session` response (or the widget
   it returns) carries a `…/return` structure, the Host **MUST**
   render a prominent return affordance (button/banner) with the
   Initiator's name.
2. The Host **MUST** validate `returnUri` against the `iss`'s
   registered `returnUri` prefixes from its well-known document
   **and** require it to be `https` or a platform-verified app link.
   Custom schemes (`myapp://`) **MUST NOT** be accepted unless
   platform-attested — this is the open-redirect lesson from OAuth
   applied directly.
3. The Host **MAY** auto-navigate to the return URI without a user
   gesture only when *all* of: the platform permits it, the user has
   previously enabled auto-return for this `iss`, and the session
   terminal state is `completed`. In all other cases the return is a
   user tap. The spec deliberately acknowledges the platform gesture
   requirement rather than fighting it.
4. The return navigation carries only `session_id` and terminal
   status — never result data:

   ```
   https://app.stargraph.example/dws/return?session=s_8f4b…&status=completed
   ```

   Results travel server-side (§6.5). URLs leak (history, logs,
   referrers); result payloads in return URLs are prohibited
   (**MUST NOT**).

---

## 9. Graceful Degradation / Polyfill Path

DWS is adoptable incrementally. Each rung down loses a property but
the flow still functions:

| Host support | Launch | Session binding | Completion | Abandonment signal | Focus return |
|---|---|---|---|---|---|
| Full Profile A + B | Signed envelope | `_meta` | Reserved tool + notifications | `session/closed` + timeout | Validated affordance / auto-return |
| Profile A only | `?q=` with embedded `session_id` | `_meta` | Reserved tool + notifications | notifications + timeout | Model-emitted link |
| Neither (today's ChatGPT/Claude) | `?q=` with embedded `session_id` | `session_id` as tool argument, per server instructions | Reserved tool by convention | Timeout only | Model-emitted link or widget `openExternal` button |

Serving the bottom row today with the same server that will serve the
top row later is an explicit design goal: the reserved tool name,
result schema, and terminal-state model are identical across rows.
This also defines the **prototype implementation** required for SEP
acceptance (§12).

---

## 10. Security Considerations

### 10.1 Return-URI validation (open redirect)

The Host renders and navigates to a URI whose ultimate source is a
third party. All lessons of [RFC 9700] apply: exact-prefix matching
against the `iss`'s pre-registered list, `https`/app-link only, no
custom schemes without platform attestation, no user-controlled
components in the validated prefix. A Host that skips this becomes a
phishing trampoline with an AI assistant's credibility attached.

### 10.2 Envelope integrity and Initiator authentication

Unsigned launch context (today's `?q=`) lets anyone craft a link that
impersonates an app and preloads a hostile prompt. The JWS envelope +
JWKS discovery gives the Host a verifiable `iss`. Hosts **MUST**
display the verified origin — not the self-declared app name alone —
at consent.

### 10.3 Replay protection

`jti`/`sessionId` are single-attach (§6.1). A replayed envelope after
attach **MUST** fail; a replayed envelope after expiry **MUST** fail on
`exp`. Servers **MUST** treat `session_id` as a bearer secret until
terminal: anyone holding an active `session_id` can bind tool calls to
the session and forge a completion. For higher-assurance deployments,
Servers **MAY** additionally bind the session to the MCP authorization
identity established at connect time and reject bound calls from other
identities.

### 10.4 Least privilege

`requestedTools` + the Server's `allowedTools` scoping (§6.2) keep the
session narrower than the Initiator's full MCP surface. Hosts **SHOULD**
surface the tool list at consent; Servers **MUST** enforce it
regardless of what the Host displays.

### 10.5 Prompt injection via launch context

`prompt` and `context.data` are attacker-influenceable inputs to a
model that holds live tool access. Mitigations, all REQUIRED of Hosts
adopting Profile B:

- Provenance-mark launch content as third-party data, not user or
  system authority (§7.3.4).
- Never grant launch content the ability to suppress the consent
  interstitial, alter the tool allowlist, or change the return URI —
  those exist only in the signed envelope, outside the text channel.
- Existing Host protections for confused-deputy tool use apply
  unchanged; DWS adds no new implicit approvals.

Servers likewise **MUST NOT** trust `dws__complete_session` arguments
beyond schema validity — `status: completed` is the model's claim, and
Initiators presenting results to users **SHOULD** label them as
assistant-produced.

### 10.6 Privacy

- Launch context transits the Host and is retained under the Host's
  data policy; Initiators **SHOULD** send references (IDs the MCP
  server can resolve) rather than raw sensitive data.
- Result data **MUST NOT** appear in return URLs (§8.4).
- `session_id` **MUST NOT** encode user identifiers.

---

## 11. Backwards and Forwards Compatibility

- DWS is a negotiated extension: non-participating clients and servers
  are unaffected.
- The extension versions independently of the MCP spec revision; the
  `version` field in the capability entry governs.
- The reserved tool name `dws__complete_session` and the `_meta` key
  are namespaced to avoid collision.
- If adoption warrants, the completion contract and session binding
  (§6) are candidates for graduation into core; the launch/return
  profile (§7–8) would remain a host-platform companion spec by
  nature.

## 12. Reference Implementation (SEP acceptance requirement)

Acceptance requires a working prototype demonstrating the proposal:

1. **Initiator app** — minimal web + Android app that creates a
   session, launches a Host, and receives the completion webhook.
2. **Reference MCP server** — implements §6 fully (attach, binding
   enforcement, reserved tool, terminal-state machine, timeouts),
   plus the degradation row of §9 against today's ChatGPT and Claude.
3. **Simulated Host** — an MCP client demonstrating Profile A
   negotiation and Profile B launch/consent/return, standing in until
   a production Host adopts the profile.
4. Conformance checklist per profile (Initiator / Server / Host).

## 13. Open Questions

1. **Multi-turn sessions:** should a session survive the user
   navigating away and returning to the same Host conversation later
   (pause/resume), or is expiry the only long-gap answer? v1 says
   expiry; resume is a candidate v1.1 addition.
2. **Host→Initiator progress events:** should the Server be able to
   push interim progress (via its own channel) with spec-blessed
   semantics, or is that purely an Initiator implementation detail?
   Current position: out of scope (§6.5 logic applies).
3. **Envelope transport ceiling:** is 16 KiB the right bound, and
   should a `contextUri` (Host fetches context from the Initiator)
   variant be normative rather than idiomatic-via-tools?
4. **Consent fatigue:** per-`iss` remembered consent vs. per-session —
   where is the OAuth "trusted app" line for agentic sessions?
5. **Extension ID:** final reverse-DNS identifier, pending sponsor
   and Working Group (likely Agents WG) input.

## 14. References

- [RFC 6749] The OAuth 2.0 Authorization Framework
- [RFC 9700] Best Current Practice for OAuth 2.0 Security
- [RFC 2119] / [RFC 8174] Key words for use in RFCs
- OIDC Core §6 — Passing Request Parameters as JWTs
- MCP Specification — capability negotiation, `_meta`, tools
- MCP SEP Guidelines — https://modelcontextprotocol.io/community/sep-guidelines
- MCP SEP-414 — trace context propagation via `_meta` (pattern precedent)
- OpenAI Apps SDK — MCP-based apps in ChatGPT (host UI + `openExternal` precedent)
- Android `ActivityResult` API — typed inter-app request/result contract

[RFC 6749]: https://www.rfc-editor.org/rfc/rfc6749
[RFC 9700]: https://www.rfc-editor.org/rfc/rfc9700
[RFC 2119]: https://www.rfc-editor.org/rfc/rfc2119
[RFC 8174]: https://www.rfc-editor.org/rfc/rfc8174
[OIDC Core §6]: https://openid.net/specs/openid-connect-core-1_0.html#JWTRequests
