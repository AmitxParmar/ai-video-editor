# MCP vs an internal tool schema for the AI video-editing agent — ecosystem state (2026)

> Research ticket: [AmitxParmar/ai-video-editor#50](https://github.com/AmitxParmar/ai-video-editor/issues/50) (parent [#1](https://github.com/AmitxParmar/ai-video-editor/issues/1)).
> Facts verified **2026-10-18** against primary sources (MCP specification, official SDK repositories, vendor documentation). Practitioner analyses are labeled as secondary at the end.

## Scope and context

The decision under review: how should the AI video-editing agent expose and execute its tools — via **MCP** (Model Context Protocol) or via an **internal function-calling/tool schema**? The agent is a backend service that turns natural-language instructions into timeline changes. Its tools are first-party and private, executed by our own API behind our own auth:

- Read-only/analysis: `generate_caption` (and media understanding tools)
- Timeline mutations: `trim_clip`, `split_clip`, `move_clip`, `add_text`, `add_audio`
- Preview/output: `preview_changes`, `render_video`

The repo's design docs already flag this as an open decision — "Tool exposure protocol (MCP vs internal function-calling schema)" — and impose cross-cutting requirements: every tool call is validated, authorized, and recorded; tool calls are one of four idempotency boundaries; approval gating is needed for mutations/render; the backend language is itself undecided (FastAPI vs Node.js). See `docs/AI Video Editor + Cloud Media Pipeline_ System Design Diagrams.md` (Diagram 8 decision points, Diagram 16) and `docs/AI Video Editor_ Design Document Set.md` (§ tool protocol row).

**Critical framing:** MCP is a *wire protocol for tool discovery and invocation* layered on top of function calling — it is not an alternative to model-facing tool schemas. Both options require JSON-Schema tool definitions passed to the LLM API and an executor that runs the call. The real question is whether our private tool layer needs a protocol boundary at all (an MCP client↔server hop over stdio or HTTP), or whether the agent service should call its own API directly with schemas held internally. The MCP spec itself describes servers offering "tools: functions for the AI model to execute" over JSON-RPC ([specification overview](https://modelcontextprotocol.io/specification)).

---

## 1. MCP in 2026: state of play

### 1.1 Specification

The current (stable) revision is **[2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/)**, released 2026-07-28 — the largest revision since launch ([Versioning](https://modelcontextprotocol.io/specification/versioning), [release post](https://blog.modelcontextprotocol.io/posts/2026-07-28/)). Revision cadence has been roughly 2–9 months: `2024-11-05` (launch) → `2025-03-26` → `2025-06-18` (introduced Streamable HTTP) → `2025-11-25` → `2026-07-28`.

Headline changes in 2026-07-28 ([full changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)):

- **Stateless core.** The `initialize` handshake and protocol-level sessions (`Mcp-Session-Id`) are gone. Every request is self-describing: protocol version, client identity, and capabilities travel in `_meta`; version mismatches return `UnsupportedProtocolVersionError`. Any request can land on any replica behind a plain round-robin load balancer.
- **`server/discover`.** A mandatory RPC advertising supported versions, capabilities, and identity — optional for clients to call up front, useful as a compatibility probe.
- **Multi Round-Trip Requests (MRTR).** Replaces server-initiated requests (sampling/elicitation/roots). A tool call can return `resultType: "input_required"` with embedded requests; the client retries with `inputResponses`. This is how mid-flight confirmations (e.g., "confirm this expensive render") work on a stateless server.
- **Tasks extension** (`io.modelcontextprotocol/tasks`, SEP-2663): poll-based `tasks/get` + `tasks/update` for long-running tool invocations. Contributed by AWS and available in Amazon Bedrock AgentCore.
- **Cacheable list results** (`ttlMs`/`cacheScope` on `tools/list` etc.) and **standard HTTP routing headers** (`Mcp-Method`, `Mcp-Name`) so gateways/WAFs can route without parsing bodies.
- **Formal deprecations**: Roots, Sampling, and Logging deprecated (SEP-2577, ≥12-month window); legacy HTTP+SSE transport deprecated; OAuth Dynamic Client Registration deprecated in favor of Client ID Metadata Documents.
- **Formal feature lifecycle and deprecation policy** with a 12-month minimum deprecation window, plus an extensions framework (Tasks, MCP Apps, Enterprise-Managed Authorization).

Backward compatibility: clients/servers detect the counterpart's era and fall back to handshake-based revisions (`2025-11-25` and earlier); `server/discover` doubles as the probe ([transports](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports), [versioning](https://modelcontextprotocol.io/specification/versioning)).

### 1.2 Governance

MCP was donated by Anthropic to the **Agentic AI Foundation (AAIF)**, a directed fund under the **Linux Foundation**, announced 2025-12-09 — co-founded by Anthropic, Block, and OpenAI with support from Google, Microsoft, AWS, Cloudflare, and Bloomberg ([Anthropic announcement](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation), [MCP blog](https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation), [Linux Foundation press release](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)). AAIF reported 188 member organizations by May 2026 ([AAIF](https://aaif.io/blog/how-to-submit-your-project-to-the-aaif)). Maintainer structure is unchanged; SPEC Enhancement Proposals (SEPs), Working Groups, SDK tiering, and conformance testing are in place ([maintainer update](https://blog.modelcontextprotocol.io/posts/2026-04-08-maintainer-update), [SDK tiering](https://modelcontextprotocol.io/community/sdk-tiers)).

### 1.3 Adoption

- "More than **10,000 published MCP servers**" at the Dec 2025 foundation announcement ([Linux Foundation](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)); there is an official registry with an API ([registry.modelcontextprotocol.io](https://registry.modelcontextprotocol.io)).
- Tier-1 SDKs see "close to **half-a-billion downloads a month**"; TypeScript and Python SDKs each crossed **1 billion total downloads** ([spec release post](https://blog.modelcontextprotocol.io/posts/2026-07-28/)).
- Checks on 2026-10-18: npm `@modelcontextprotocol/sdk` v1.32.1 (~65M downloads/week), PyPI `mcp` v2.3.0, official Ruby SDK at 1.0 ([npm](https://www.npmjs.com/package/@modelcontextprotocol/sdk), [PyPI](https://pypi.org/project/mcp/), [Ruby SDK post](https://blog.modelcontextprotocol.io/posts/ruby-sdk-1-0/)).
- Adopted by ChatGPT, Claude, Cursor, Gemini, VS Code, and Microsoft Copilot ([Anthropic](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation)); enterprise platforms ship it (Amazon Bedrock AgentCore, Cloudflare Agents SDK, Microsoft Foundry, Google Cloud, Netlify — vendor statements in the [release post](https://blog.modelcontextprotocol.io/posts/2026-07-28/)).
- Enterprise-Managed Authorization (zero-touch OAuth) extension went stable in June 2026, adopted by Anthropic, Microsoft, and Okta ([blog](https://blog.modelcontextprotocol.io/posts/enterprise-managed-auth/)).

### 1.4 Transports

MCP defines two standard transports plus custom ones ([transports spec](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports)):

| Transport | Shape | Auth guidance | Notes |
|---|---|---|---|
| **stdio** | Newline-delimited JSON-RPC over a client-launched subprocess | Spec: SHOULD NOT use the OAuth flow; "retrieve credentials from the environment" | Local tool processes; framing is reusable over UDS/TCP for custom transports |
| **Streamable HTTP** | One POST endpoint; replies as JSON or request-scoped SSE; stateless since 2026-07-28 (no sessions, no GET/DELETE stream, no resumption) | OAuth 2.1 resource-server profile | Plain HTTP workload; load-balances with round-robin; `Mcp-Method`/`Mcp-Name` headers for gateways |

The legacy HTTP+SSE transport is deprecated. Practically: a **server-side agent backend would use Streamable HTTP**, not stdio; stdio only makes sense for locally launched tool processes.

### 1.5 Auth

Authorization is **optional** in MCP but specified for HTTP transports ([authorization spec](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)):

- MCP server acts as an **OAuth 2.1 resource server**; clients obtain tokens via an authorization server discovered through OAuth 2.0 Protected Resource Metadata (RFC 9728); tokens must be audience-bound (RFC 8707 resource indicators); `iss` validation per RFC 9207; Client ID Metadata Documents preferred, Dynamic Client Registration deprecated.
- Scope guidance is explicit (least privilege, `WWW-Authenticate` scope challenges, step-up authorization).
- Security best practices document known attack classes: confused deputy, token passthrough, SSRF during metadata discovery, state-handle hijacking (new with the stateless core), scope inflation, local-server compromise ([security best practices](https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices)).
- The maintainers state plainly: "**authorization is where implementers spend most of their integration time**" ([release post](https://blog.modelcontextprotocol.io/posts/2026-07-28/)).

For a *private* tool layer, this is the single biggest cost driver — and the least necessary component if the only MCP client is our own agent backend (which already authenticates to our API with first-party credentials).

### 1.6 Rust SDK: rmcp

| Fact | Value | Source |
|---|---|---|
| Crate | `rmcp` (official Rust SDK, modelcontextprotocol org) | [GitHub](https://github.com/modelcontextprotocol/rust-sdk) |
| Latest release | 3.5.1 on 2026-10-05 (3.x line since 2026-07-28) | [crates.io](https://crates.io/crates/rmcp), [releases](https://github.com/modelcontextprotocol/rust-sdk/releases) |
| Total downloads | ~37M (≈20.6M recent) | crates.io API |
| Maintenance | Active: repo pushed 2026-10-17; 4k stars; steady releases (3.1→3.5 across Aug–Oct 2026) | GitHub API |
| Spec support | Implements stable `2026-07-28`, backward compatible with `2025-11-25` and earlier; `ClientLifecycleMode::Auto` probes discovery and falls back | [README](https://github.com/modelcontextprotocol/rust-sdk) |
| Feature depth | `#[tool]`/`#[tool_router]` macros with `schemars` JSON Schema 2020-12; stdio + Streamable HTTP server/client; **stateless Streamable HTTP by default** (plain-JSON replies option); Tasks (task manager); MRTR with HMAC-sealed `requestState`; response caching; subscriptions; progress/cancellation; OAuth support (docs/OAUTH_SUPPORT.md) | README |
| Integration shape | Server is a Tower service — mountable on axum (`nest_service("/mcp", ...)`); fresh handler per request in stateless mode (keep shared state in `Clone` handles) | README |

Caveat on SDK tier: the four **Tier-1 SDKs are TypeScript, Python, Go, and C#**; the July 2026 release post said the Rust SDK supported the new spec "in beta", and by October rmcp 3.5.1 states it implements the stable `2026-07-28` spec with release cadence roughly weekly-to-monthly ([SDK tiering](https://modelcontextprotocol.io/community/sdk-tiers), [release post](https://blog.modelcontextprotocol.io/posts/2026-07-28/)). Rust support is therefore real and production-usable but a notch behind Tier 1 on formal conformance guarantees. If the backend lands on Node.js instead, the TS SDK is Tier 1 and similarly mature.

### 1.7 Direction of travel (roadmap, Aug 2026)

Five priority areas ([roadmap post](https://blog.modelcontextprotocol.io/posts/mcp-roadmap/)): agentic messaging primitives (server-initiated events/webhooks, maturing Tasks into the spec), HTTP-native transport unification, agent identity/enterprise security (DPoP, Workload Identity Federation, token exchange), improved primitives (**progressive discovery** for large tool catalogs, one clear result contract), and SDK developer experience. Notably for us: the roadmap acknowledges "connecting to a server with a hundred tools means the model pays for that entire surface" — ours is ~8–10 tools, so catalog scale is a non-issue.

Risk read: the protocol is still actively changing (breaking changes landed as recently as 2026-07-28), but it now has neutral governance, a deprecation policy, conformance tests, and wide vendor backing. The main churn risk for an early adopter is betting on optional/deprecated features — sampling, roots, and logging are already deprecated; Tasks, MCP Apps, and EMA are extensions outside the core.

---

## 2. The internal function-calling option

Model APIs consume JSON-Schema tool definitions and return structured calls that the application executes:

- **OpenAI** ([function calling guide](https://platform.openai.com/docs/guides/function-calling)): functions declared in the `tools` parameter with `strict` mode, `tool_choice` (auto/required/forced/allowed lists), parallel calls, namespaces, deferred loading via tool search; execution is application-side ("Execute code on the application side with input from the tool call").
- **Anthropic** ([tool use overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)): client tools via `input_schema`, `tool_use`/`tool_result` round trip, `strict: true` schema conformance, server-side tool search for large catalogs.

For our architecture this means: the agent service holds a **tool registry** (name, description, JSON Schema, handler, authz rules, idempotency semantics), serializes the schemas into whichever provider API we call, receives a tool call, validates it, executes it **directly against our own API layer** (in-process call or internal HTTP), records it, and returns the result to the model. No additional wire protocol, no second schema authority, no MCP server to operate.

This is the layer MCP would sit on top of. Choosing "internal" is not a bet against MCP; it is a bet that — for the first version — the protocol's core benefits (runtime discovery, cross-vendor interop, remote invocation) have no consumer other than our own agent.

---

## 3. Comparison table

| Dimension | MCP (2026-07-28) | Internal function-calling schema |
|---|---|---|
| What it is | Open wire protocol (JSON-RPC 2.0) for discovery + invocation of tools/resources/prompts, layered **on top of** function calling | Tool definitions passed directly to the LLM API; execution in our code |
| Tool schema format | JSON Schema 2020-12 `inputSchema`/`outputSchema`, same as function calling | JSON Schema (vendor-specific wrappers: OpenAI `parameters`, Anthropic `input_schema`) |
| Discovery | Runtime `tools/list` (+ `server/discover`, list-changed notifications, cacheable results) | Compile-time: our registry is the source of truth |
| Execution path | Agent → MCP client → (stdio subprocess \| Streamable HTTP POST) → MCP server → our API | Agent → tool handler → our API (one hop removed) |
| Transports | stdio; Streamable HTTP (stateless since 2026-07-28); custom (UDS/TCP allowed) | Whatever the agent uses internally (direct call or internal HTTP) |
| Auth model | OAuth 2.1 resource server for HTTP (PRM/RFC 9728, resource indicators/RFC 8707, `iss`, CIMD; EMA extension); env credentials for stdio; security best practices doc | Reuse existing API auth (service credentials, user context); authorization logic already required by our design docs |
| Latency / hops | Adds a process or network hop; spec-side mitigations: stateless round-robin, cacheable lists, header routing | In-process or single internal call — lowest latency and failure surface |
| Long-running operations | Tasks extension: poll-based `tasks/get`, mid-flight `tasks/update`, cooperative cancel | Our own job queue already handles async render jobs (an existing idempotency boundary) |
| Mid-flight input / approvals | MRTR: `input_required` + retry with `inputResponses` — good for external clients confirming expensive actions | Our own approval-gating flow (design docs: gate mutations/`render_video`) |
| Progress / cancellation | Standard `notifications/progress` + cancellation semantics | Home-grown (or already covered by our job/streaming layer) |
| Caching | `ttlMs`/`cacheScope` on list/read results (SEP-2549) | Our own caching; prompt-cache-friendly deterministic schema order is trivially ours to control |
| Versioning risk | Spec revisions every few months; breaking change in 2026-07-28; formal deprecation policy (12-month window); deprecated: sampling, roots, logging, HTTP+SSE, DCR | Vendor API churn instead (OpenAI Responses vs Chat Completions, Anthropic versions); generally additive |
| Multi-provider stance | Provider-neutral by design; one server works across hosts | Schemas ~95% portable; execution and call plumbing ours anyway |
| Ecosystem interop | The point of MCP: plugs into Claude, ChatGPT, Cursor, VS Code, Copilot, gateways, registries; consumable servers are plentiful (10k+) | None beyond our own product |
| Rust support | `rmcp` 3.5.1 official, active, deep feature coverage incl. Tasks/MRTR; not a Tier-1 SDK (TS/Python/Go/C# are) | No SDK needed — plain serde/schemars JSON |
| Operational cost | Extra service/process to build, deploy, monitor, version-pin; OAuth if exposed to non-first-party clients | None beyond the tool registry we already need |
| Security surface | Adds protocol-level attack classes (confused deputy, token passthrough, SSRF on metadata discovery, state-handle hijacking); mostly relevant when either side is not first-party | Conventional API security; our existing validation/authz/audit applies |
| Fit for a private tool layer behind our own API | Value unlocks only with interop (external hosts, third-party tools, multi-host reuse) | Exactly the shape of our current requirement |

---

## 4. Fit for our system

**What MCP would buy us that we don't already have:**

1. **External tool hosting.** If Claude Desktop / Cursor / VS Code / ChatGPT connectors or third-party products should one day drive the editor's tools, one MCP server exposes them all. This is the strongest long-term reason.
2. **A standard plugin boundary.** If third parties (or other teams) contribute tools/effects, MCP is a ready-made contract with discovery, versioned capabilities, and isolation.
3. **Consuming third-party MCP servers.** Stock footage, transcription, media-analysis tools already speak MCP. That's a *client-side* reason — relevant to the agent even if our own tools stay internal.
4. **Standard long-running + approval semantics for external clients.** Tasks (polling, mid-flight update) and MRTR (`input_required` confirmations) map well to `render_video` costs and mutation approvals — *if the caller is an external MCP client that understands them*. Our own web UI/agent loop already implements approval gating and jobs.

**What MCP would cost us now:** a second interface to build and keep spec-current (even stateless, it's a service with release discipline), OAuth integration time if anything but our own backend calls it (maintainers call auth the biggest integration cost), a schema/dispatch layer that duplicates the registry we must own anyway, and exposure of the *model* to protocol churn (e.g., don't build on deprecated sampling/roots/logging; pin a revision).

**Requirements check (from the design docs):**

| Requirement | Internal schema | MCP 2026-07-28 |
|---|---|---|
| Validate/authorize/record every tool call | Registry + existing API authz/audit | Same, plus optional OAuth for non-first-party clients |
| Idempotency for tool calls | Idempotency keys in the registry | Same; stateless spec also recommends explicit, server-minted handles (e.g., a render/job ID) as tool arguments — compatible |
| Approval gating for mutations/render | Our approval flow | MRTR `input_required` (useful for external clients; our own flow still needed for the UI) |
| Long-running renders | Existing job queue + render job idempotency boundary | Tasks extension (polling/update) if external clients should drive renders |
| Provider abstraction (OpenAI/Anthropic/…) | Registry serializes per provider | MCP is provider-neutral, but the agent still needs provider tool-schema projection regardless |

Read: **MCP solves coordination problems (many clients, many servers, untrusted third parties) that our current system does not have.** Its 2026 direction — stateless HTTP, header routing, Tasks, MRTR, registry — makes it excellent *when those problems arrive*, and cheaper than ever to adopt late.

---

## 5. Bottom line for the decision

**Recommended default: build the v1 tool layer as an internal function-calling schema behind a single protocol-agnostic tool registry, executed directly against our own API. Treat MCP as a thin, reversible facade to add when an interop trigger fires — not as the transport for our own backend↔own-API call path.**

Rationale:

1. **The only MCP client today would be our own agent, and the only server our own API.** MCP's core value (discovery, cross-vendor interop, remote invocation) is consumed when client and server are independently governed. Ours are not; the protocol hop is pure overhead, and its OAuth profile addresses a threat model (third-party clients) we don't have yet.
2. **The expensive parts of MCP are auth and version discipline, not the protocol shape.** The spec maintainers say authorization dominates integration effort; and the spec moved breakingly as recently as July 2026. Keeping the wire protocol out of v1 means we don't pay either cost until we get the interop benefit.
3. **The SDK risk is no longer a blocker.** `rmcp` 3.5.1 (Oct 2026) is official, actively maintained, and implements the stable 2026-07-28 spec including Tasks and MRTR; if the backend lands on Node.js, the Tier-1 TS SDK is even safer. "MCP later" is a small adapter, not a rewrite.
4. **Deferring costs us almost nothing.** The 2026-07-28 stateless core makes an MCP server a plain HTTP workload (round-robin, no sessions), and rmcp mounts on axum as a Tower service. A future facade is a thin translation over the same registry.

**Design guardrails that keep the MCP option cheap (do these now):**

- One **tool registry** as the single source of truth: name, description, JSON Schema, handler, authz rules, idempotency key, risk class (read-only / mutation / expensive). Nothing downstream may define tools a second time.
- Keep handlers **protocol-free** — they take validated arguments and call our API; MCP/function-calling layers only adapt schema and dispatch.
- Adopt MCP-shaped **metadata conventions** now: stable names, descriptions, and annotations for read-only/destructive/idempotent behavior (MCP tool annotations; also matches our approval-gating and idempotency decisions).
- Model long-running work as **explicit handles** (render/job IDs passed as tool arguments). This matches our idempotency boundaries *and* the stateless MCP guidance, and maps 1:1 onto the Tasks extension later.
- Avoid betting on deprecated MCP features (sampling, roots, logging) and on protocol-level sessions in any design discussions; if/when adopting MCP, pin the revision and plan for the 12-month deprecation windows.

**Triggers that should flip the decision (add the MCP facade over the same registry, using rmcp, Streamable HTTP, stateless):**

1. External MCP hosts (Claude/ChatGPT/IDE/Copilot connectors) must drive editor tools.
2. Third-party tools or plugin-style contributions become part of the product.
3. The agent needs to **consume** third-party MCP servers (stock media, transcription, analysis) — the client side of MCP, which is independent of exposing our tools.
4. A second agent host or service appears (multiple internal consumers of the same tools).
5. External clients need standardized render tasks/cancellation (Tasks) or mid-flight cost confirmations (MRTR).

If any of 1–3 is already committed for v1, invert the default: build the MCP facade and client from day one with rmcp 3.x (Rust) or the Tier-1 TS SDK (Node), and only then take on the OAuth profile for non-first-party clients.

**Do not:** put a protocol boundary between the agent and our own API for v1; adopt deprecated MCP features; or let a second tool-schema source of truth emerge alongside the registry.

---

## Sources

**MCP specification (primary)**

- Specification index / current revision — https://modelcontextprotocol.io/specification
- 2026-07-28 changelog — https://modelcontextprotocol.io/specification/2026-07-28/changelog
- Versioning and revisions — https://modelcontextprotocol.io/specification/versioning
- Transports (stdio, Streamable HTTP, custom) — https://modelcontextprotocol.io/specification/2026-07-28/basic/transports
- Authorization — https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization
- Security best practices — https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices
- SDK tiering system — https://modelcontextprotocol.io/community/sdk-tiers

**MCP blog and governance (primary)**

- The 2026-07-28 Specification — https://blog.modelcontextprotocol.io/posts/2026-07-28/
- The New MCP Roadmap (Aug 2026) — https://blog.modelcontextprotocol.io/posts/mcp-roadmap/
- MCP joins the Agentic AI Foundation — https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation
- Enterprise-Managed Authorization extension stable — https://blog.modelcontextprotocol.io/posts/enterprise-managed-auth/
- Maintainer team update — https://blog.modelcontextprotocol.io/posts/2026-04-08-maintainer-update
- Ruby SDK 1.0 — https://blog.modelcontextprotocol.io/posts/ruby-sdk-1-0/
- Anthropic donation announcement — https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation
- Linux Foundation press release (AAIF, 10,000+ servers) — https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation
- AAIF project submission post (188 members) — https://aaif.io/blog/how-to-submit-your-project-to-the-aaif

**Rust SDK (rmcp) and SDK registries (primary)**

- rmcp README and docs — https://github.com/modelcontextprotocol/rust-sdk
- crates.io entry (release history, downloads) — https://crates.io/crates/rmcp
- npm TypeScript SDK — https://www.npmjs.com/package/@modelcontextprotocol/sdk
- PyPI Python SDK — https://pypi.org/project/mcp/
- Official MCP server registry — https://registry.modelcontextprotocol.io

**Vendor tool-calling docs (primary)**

- OpenAI function calling — https://platform.openai.com/docs/guides/function-calling
- Anthropic tool use — https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview

**Practitioner analyses (secondary; used only for framing, not facts)**

- OpenClaw, "MCP vs Function Calling: When to Use Which in 2026" — https://openclaw-ai.net/en/blog/mcp-vs-function-calling-2026
- Portkey, "MCP vs Function Calling – How They Actually Work Together" — https://portkey.ai/blog/mcp-vs-function-calling
- Nango, "MCP vs tool calls for AI agents" (Jul 2026) — https://nango.dev/blog/mcp-vs-tool-calls-for-ai-agents
- Prefect, "MCP vs Function Calling: When to Use Which" — https://www.prefect.io/resources/mcp-vs-function-calling

**Repository context**

- `docs/AI Video Editor + Cloud Media Pipeline_ System Design Diagrams.md` (Diagram 8: agent tool catalog and decision points; Diagram 16)
- `docs/AI Video Editor_ Design Document Set.md` (tool protocol row; validation/authorization/audit and idempotency requirements)
