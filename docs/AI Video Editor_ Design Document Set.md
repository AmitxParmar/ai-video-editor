# AI Video Editor + Cloud Media Pipeline: Design Document Set

Seven documents that expand the 20-diagram set. Same conventions as before: no code and no implementation steps, and anything not settled is marked **Decision Point**. Where a document proposes something (for example MVP scope or performance targets), it is labelled **Proposal** so you can accept, change, or reject it.

**Contents**

1. PRD / MVP Scope
2. System Architecture
3. Browser Video Editor Architecture
4. Media + Rendering Pipeline
5. AI Editing Agent Design
6. Performance
7. Reliability

---

# 1. PRD / MVP Scope

## 1.1 Problem statement

Video editing is slow for people who know what they want but not how to execute it in a timeline tool. Desktop editors need heavy installs and large local machines. Browser editors exist but struggle with large media. An AI agent that can understand footage and make edits, with the user in control, shortens the path from intent to finished video.

## 1.2 Product vision

A browser-based editor where large source media lives in the cloud, editing feels responsive through proxy media, and an AI agent can propose and apply edits that the user reviews before anything is final.

## 1.3 Target users

**Decision Point:** primary persona. Candidates: solo creators (short-form social), small marketing teams, educators/course creators, podcasters repurposing video. The persona drives codec support, collaboration needs, and the agent's default behaviours.

## 1.4 Goals and non-goals

**Goals**

- Upload large videos reliably from the browser.
- Edit on a multi-track timeline with responsive preview.
- Instruct an AI agent in natural language and review its proposed edits.
- Export a final video rendered in the cloud.
- Never lose user work.

**Non-goals (Proposal, for first release)**

- Professional colour grading, VFX compositing, motion graphics authoring.
- Native mobile apps.
- Live streaming or live capture.
- Marketplace of effects/templates.

## 1.5 User stories (grouped)

| Area | Story |
| --- | --- |
| Projects | As a user I create, open, rename, and delete projects. |
| Upload | As a user I upload large files and resume if my connection drops. |
| Library | As a user I see my media with thumbnails, duration, and processing status. |
| Editing | As a user I trim, split, move, and delete clips on a timeline with multiple tracks. |
| Overlays | As a user I add text and background audio. |
| Preview | As a user I play back my timeline smoothly and scrub quickly. |
| AI | As a user I tell the agent what I want, see what it proposes, and approve or reject. |
| Export | As a user I export my project and download or share the result. |
| Recovery | As a user I return after a crash or disconnect and find my work intact. |

## 1.5b Proposed MVP scope

**Proposal** (each row is a scope Decision Point):

| Capability | MVP | Later |
| --- | --- | --- |
| Auth, single-user projects | In | Teams/workspaces |
| Chunked resumable upload | In | Import from URL / cloud drives |
| Proxy, thumbnail, waveform generation | In | Multiple proxy renditions |
| Multi-track timeline: trim, split, move | In | Ripple/roll edits, keyframes |
| Text overlays, one audio track | In | Rich text styling, audio mixing |
| Transitions and effects | Minimal set | Extensible library |
| Cloud render to a single output format | In | Multiple presets, resolutions |
| Agent: instruction, plan, proposed edits, approval | In | Multi-step autonomous runs |
| Agent: transcript search, scene detection | In | Visual semantic search |
| Auto-captioning | In or Later (Decision Point) |  |
| Offline editing with sync | Crash recovery only | Full offline-first |
| Real-time multi-user collaboration | Out | Later |
| Billing and quotas | Basic limits | Full metering |

## 1.6 Functional requirements (summary)

- FR1: Authenticated users own projects and assets; access is private by default.
- FR2: Upload supports large files, pause/resume, and shows state.
- FR3: Each uploaded asset reaches a clear status (processing, ready, failed) with reasons.
- FR4: The timeline persists automatically and survives refresh and crash.
- FR5: The agent never alters the final project without user approval (default policy, see Decision Point in doc 5).
- FR6: Render jobs are asynchronous, report progress, are cancellable, and are retried on transient failure.
- FR7: Outputs are delivered through time-limited access.

## 1.7 Non-functional requirements (summary)

Responsiveness, durability of user work, privacy of media, cost control for rendering and AI. Detailed in docs 6 and 7.

## 1.8 Success metrics

**Proposal:** upload completion rate, time from upload to editable, render success rate, median render time per minute of output, percentage of agent proposals accepted, edits per session, crash-recovery success rate.

## 1.9 Constraints and assumptions

- Browser capability varies; supported browsers must be declared. (**Decision Point:** minimum browser matrix.)
- Source media may be large and in varied codecs and containers.
- External AI providers are used; media content may leave the platform.
- Portfolio/interview context: documentation quality and clear boundaries matter as much as feature count.

## 1.10 Risks

| Risk | Impact |
| --- | --- |
| Browser memory and decode limits on large media | Editing becomes unusable |
| Render cost and queue backlog | Cost overruns, slow exports |
| AI makes wrong or destructive edits | Loss of trust |
| Provider outage or policy changes | Agent unavailable |
| Scope creep toward a full professional editor | Nothing finishes |

## 1.11 Open questions

Primary persona, auto-captioning in MVP, minimum browser support, whether free-tier limits exist, data retention period for originals and renders.

---

# 2. System Architecture

## 2.1 Architectural principles

1. **Heavy media never passes through the API.** Upload and delivery go directly to object storage and CDN.
2. **Long-running work is asynchronous.** Anything that can exceed a request lifetime goes through the queue.
3. **A durable source of truth for every concept.** Each piece of data has exactly one owner.
4. **The agent is a client of the system, not a privileged part of it.** It acts through the same tool boundary and permission checks as users.
5. **Humans approve consequential outcomes.**
6. **Few deployables.** Prefer modules with clear boundaries over many services.

## 2.2 Logical components

| Component | Responsibility | Owns |
| --- | --- | --- |
| Web Editor | Interaction, preview, local state | Local working copy |
| API Service | Authentication, authorization, request routing | Nothing durable |
| Project/Timeline Service | Project and timeline CRUD, validation of edits | Projects, timelines, tracks, clips |
| Media Service | Asset registry, status, derived-artifact references | Assets |
| Upload Service | Upload session control and state | Upload sessions |
| Metadata/Analysis Service | Technical metadata, transcripts, scene data | Analysis results |
| Render Orchestrator | Render job lifecycle, progress, retry | Render jobs |
| Workers (ingest, render) | Execute media jobs | Nothing durable except outputs |
| AI Agent Service | Agent runs, planning loop, provider calls | Agent runs, tool-call records |
| Agent Tool Layer | Controlled, permissioned operations for the agent | Nothing |
| Queue | Decouple producers from workers | Pending work |
| PostgreSQL | System of record for structured data | Structured data |
| Redis | Ephemeral state, pub/sub, caching | Nothing authoritative |
| Object Storage | Media bytes: originals, derived, outputs | Media bytes |
| CDN | Delivery of media | Cached copies |
| Observability | Logs, metrics, traces | Telemetry |

**Decision Point:** deployment form of the six backend components: one modular backend, a few services, or separate services. Boundaries stay the same either way.

## 2.3 Key flows

- **Upload:** browser to object storage directly, with Upload Service tracking state; a completion event triggers the media pipeline.
- **Edit:** browser mutations go to Project/Timeline Service; changes fan out through the real-time channel.
- **Agent run:** instruction to Agent Service, which uses tools; proposed changes validated, then approved by the user.
- **Render:** snapshot of timeline, queue, worker, output in storage, event to browser.

## 2.4 Data ownership and consistency

- Structured data: strong consistency inside PostgreSQL.
- Media bytes: object storage with immutable, versioned keys per artifact.
- Cross-boundary updates (upload completion, render completion) are event-driven and must tolerate duplicate delivery.
- **Decision Point:** event publication guarantee between database writes and queue messages (same-transaction pattern vs reconciliation).

## 2.5 Synchronous vs asynchronous boundary

Synchronous: authentication, project reads/writes, signed URL issuance, agent instruction intake. Asynchronous: ingest, analysis, render, agent execution, progress events.

## 2.6 Cross-cutting concerns

Security (doc: Diagram 16), observability (Diagram 17), correlation identifiers across all hops, multi-tenancy scoping on every data access.

## 2.7 Quality attributes ranking

**Proposal:** (1) Durability of user work, (2) Editing responsiveness, (3) Render correctness, (4) Cost control, (5) Agent usefulness, (6) Time-to-market. Ranking is a Decision Point because it settles trade-offs.

## 2.8 Architecture decisions pending

| Topic | Options to evaluate |
| --- | --- |
| Backend runtime | FastAPI or Node.js |
| Queue technology | Managed queue, Redis-based, log-based |
| Timeline storage | Normalized rows or versioned document |
| Real-time transport | WebSocket or SSE |
| Tool protocol | MCP or internal schema |
| Tenancy model | Shared schema with scoping, or stronger isolation |

---

# 3. Browser Video Editor Architecture

## 3.1 Responsibilities

The editor lets the user compose a timeline, preview it with low latency, manage media, converse with the agent, and keep work safe locally while syncing it to the server. It does not render final output (default; see Decision Point 3.10).

## 3.2 Modules

| Module | Responsibility |
| --- | --- |
| Timeline UI | Tracks, clips, playhead, selection, zoom |
| Preview Player | Compose and play the current timeline |
| Media Library | Assets, status, thumbnails, upload entry points |
| Agent Panel | Instruction input, plan view, proposed-change review, approve/reject |
| Editor State | Single in-memory model of project, selection, playback |
| Command Log | Ordered record of user and agent mutations |
| Persistence | Local durable copy of state and pending changes |
| Sync Layer | Exchange mutations and events with the backend |
| Media Workers | Decode, thumbnails, waveforms, upload chunking |

## 3.3 State ownership

- **Project state:** the document being edited, mirrored with the server.
- **Ephemeral UI state:** selection, zoom, hover; never synced.
- **Playback state:** playhead and play status; local only.
- **Pending state:** mutations not yet acknowledged by the server.

**Decision Point:** project state as a command log, as snapshots, or both. This determines undo/redo, sync granularity, and recovery.

## 3.4 Threading boundary

The main thread handles interaction and drawing. Workers handle decoding, analysis, hashing, and upload. Communication is message-based, so state crossing the boundary must be serializable. **Decision Point:** whether workers hold their own copies of state or receive read-only slices.

## 3.5 Media handling

- Editing uses proxy media streamed from the CDN, not originals.
- Local originals the user has chosen may be used for immediate preview before cloud processing finishes. **Decision Point:** whether to support this.
- Decoded frames are held in bounded windows around the playhead.

## 3.6 Preview model

**Decision Point (major):** how preview is produced.

- Client-side composition of proxy streams (responsive, limited by device).
- Server-rendered preview segments (consistent, higher latency and cost).
- Hybrid: client for simple timelines, server for complex ones.

## 3.7 Local persistence

Local store holds the latest state, the command log, pending mutations, and cached metadata. It serves crash recovery and short offline periods. See doc 7 and Diagram 12.

## 3.8 Network synchronization

Mutations are identified, ordered, and acknowledged. The browser tolerates duplicate acknowledgements. Conflict policy depends on the collaboration Decision Point (single editor vs multi-user).

## 3.9 Agent interaction in the browser

Agent proposals arrive as structured change sets. The editor renders them as a reviewable diff against the current timeline and applies them only on approval. **Decision Point:** whether proposals are applied to a draft branch of the timeline or kept as an overlay.

## 3.10 Out of scope in the browser (default)

Final export. **Decision Point:** whether small projects can be exported client-side.

## 3.11 Constraints

Browser memory limits, codec support variance, tab backgrounding that throttles timers and workers, storage quota limits.

---

# 4. Media + Rendering Pipeline

## 4.1 Purpose

Turn uploaded files into editor-ready artifacts, and turn timeline definitions into final videos.

## 4.2 Artifact catalogue

| Artifact | Produced by | Used for | Lifetime |
| --- | --- | --- | --- |
| Original | Upload | Source of truth for render | Retained per policy |
| Technical metadata | Ingest | Validation, timeline math, agent context | With asset |
| Proxy | Ingest | Editing and preview | Regenerable |
| Thumbnails | Ingest | Library and timeline strip | Regenerable |
| Waveform | Ingest | Audio display | Regenerable |
| Transcript | Analysis | Search, captions, agent | With asset |
| Scene boundaries | Analysis | Agent, navigation | With asset |
| Intermediate render artifacts | Render | Segment stitching, resume | Short-lived |
| Final output | Render | Delivery | Retained per policy |

## 4.3 Asset lifecycle states

`UPLOADING → UPLOADED → VALIDATING → PROCESSING → READY`, with `REJECTED` (failed validation) and `FAILED` (processing error, retryable). Partial readiness is possible (proxy ready before waveform), so readiness is per-artifact.

**Decision Point:** is an asset usable on the timeline when the proxy is ready, or only when everything is ready.

## 4.4 Ingest stages

Validation (is it real, supported, within limits), metadata extraction, transcoding to a working format, then parallel derivations (proxy, thumbnails, waveform), then optional analysis (transcript, scenes). Each stage is independently retryable and idempotent on its output key.

**Decision Point:** analysis is eager at ingest, lazy on first agent use, or user-triggered. This is a major cost lever.

## 4.5 Render model

- Input: an immutable snapshot of the timeline plus references to original media.
- Orchestrator: validates, plans, schedules, tracks progress, handles retry.
- Worker: executes the plan using the media engine, writing to storage.
- Output: final file plus a completion event.
- Cancellation: a render can be cancelled; workers observe cancellation between units of work.

**Decision Points**

- Whole-job vs segmented rendering with stitching.
- Rendering from originals vs a mezzanine format.
- Output presets for MVP.
- Preview renders vs final renders as separate job types.

## 4.6 Job model

Each job has: id, type, owner, input references, state, attempts, timestamps, progress, result reference, and error classification. States: `QUEUED, RUNNING, SUCCEEDED, FAILED, CANCELLED`, plus `RETRYING` or equivalent. Job records are durable in PostgreSQL, independent of queue message state.

## 4.7 Error classification

| Class | Examples | Handling |
| --- | --- | --- |
| Transient | Worker lost, storage timeout | Retry |
| Input defect | Corrupt file, unsupported codec | Fail with user-visible reason |
| Resource | Out of memory, disk full | Retry on larger class or fail |
| Policy | Over quota, cancelled | Stop, report |

## 4.8 Delivery

Final outputs and derived artifacts are delivered via CDN with time-limited access. Progress is delivered by events, not polling (default).

## 4.9 Open questions

Maximum supported file size and duration, codec/container support list, audio-only assets, retention windows, and whether users can re-download originals.

---

# 5. AI Editing Agent Design

## 5.1 Purpose and scope

The agent turns natural-language instructions into proposed timeline changes, using understanding of the media. It assists; it does not silently publish.

**In scope (Proposal):** find moments, trim/split/move clips, add text, add audio, generate captions, preview and request render. **Out of scope:** generating new video, editing without the user's project context, acting across projects.

## 5.2 Design principles

1. All effects go through tools. The agent has no direct database or storage access.
2. Reads are cheap and free-flowing; mutations are explicit and traceable.
3. Every tool call is validated, authorized, and recorded.
4. The user sees what will change before it becomes real.
5. The agent is bounded: step limits, time limits, cost limits.

## 5.3 Run lifecycle

CREATED, PLANNING, EXECUTING, WAITING_FOR_TOOL, VALIDATING, WAITING_FOR_APPROVAL, RENDERING, COMPLETED, FAILED, CANCELLED (see Diagram 9). Each transition is durable so a run can be inspected and, where chosen, resumed.

## 5.4 Context available to the agent

Project summary, timeline structure, asset metadata, transcript and scene data, the user's instruction and conversation history for the run. **Decision Point:** how much of this is provided up front vs fetched through tools (affects cost and quality).

## 5.5 Tool catalogue

| Class | Tools | Side effects | Needs approval |
| --- | --- | --- | --- |
| Inspect | inspect_project, inspect_media, search_transcript, find_clip | None | No |
| Analyze | detect_scenes, generate_caption | Compute and AI cost; caption creates content | No / Decision Point |
| Mutate | trim_clip, split_clip, move_clip, add_text, add_audio | Changes proposed timeline | At change-set level |
| Preview | preview_changes | Compute | No |
| Output | render_video | Significant cost | Yes |

Each tool has a declared input contract, output contract, permission requirement, cost class, and idempotency behaviour.

## 5.6 Planner and executor boundary

**Decision Point (major):** single loop (model decides next tool each step), plan-then-execute, or multiple specialised agents. This affects predictability, cost, and the quality of approval UX.

## 5.7 Validation

Before proposals reach the user, a validator checks structural integrity (no overlaps where disallowed, clip bounds within source duration, referenced assets exist and are ready), policy (limits), and sanity (the result matches the instruction scope). Failures return to the planner as feedback, bounded by retry limits.

## 5.8 Human approval

Default **Proposal:** all mutations appear as a reviewable change set; the user can accept all, accept parts, or reject with feedback; rendering requires explicit approval. **Decision Point:** per-user autonomy levels (always ask, ask before render only, auto-apply small edits).

## 5.9 Model provider abstraction

A narrow internal interface isolates providers: capabilities needed (text reasoning, tool use, transcription, vision). **Decision Points:** number of providers at launch, fallback policy, which tasks use which model class, and how provider-specific features are handled.

## 5.10 Safety and privacy

Prompt injection from media content (for example text in transcripts or on-screen) must not grant new capabilities; tool permissions are fixed by the system, not by content. User media sent to providers is a privacy boundary requiring disclosure and policy. **Decision Point:** per-project opt-in for external analysis.

## 5.11 Cost controls

Budgets per run and per user, caps on tool calls and analysis scope, caching of analysis results, preference for cheaper models on routine steps.

## 5.12 Evaluation

**Proposal:** a fixed set of representative instructions and projects, checked for correctness of resulting timelines, validator pass rate, approval rate, steps and cost per run, and failure modes. Evaluation criteria are defined before building so quality claims are testable.

## 5.13 Observability

Every run has a trace: instruction, plan, each tool call with inputs/outputs summaries, validation results, user decision, cost. Traces may contain user content, so retention and access are governed (Security doc, Diagram 16).

---

# 6. Performance

## 6.1 Scope

Responsiveness in the browser, throughput of the media pipeline, latency and cost of the agent, and delivery speed.

## 6.2 Performance dimensions

| Dimension | What matters |
| --- | --- |
| Interaction | Input latency in timeline operations, no main-thread stalls |
| Playback | Smooth preview, quick start, quick seek |
| Upload | Throughput and resumability for large files |
| Ingest | Time from upload complete to editable |
| Render | Time per output minute, queue wait |
| Agent | Time to first proposal, total run time |
| Delivery | Time to first frame from CDN |

## 6.3 Proposed targets

**Proposal, to be validated with measurement:** each dimension gets a latency/throughput budget, defined at a percentile (for example p50 and p95), measured in real browsers on a defined reference device class. Numbers are intentionally not fixed here; setting them is a Decision Point tied to persona and device assumptions.

## 6.4 Where pressure comes from

- Large files: memory, bandwidth, transcode time.
- Many clips and tracks: rendering the timeline UI and preview composition.
- Concurrent renders: compute contention and queue wait.
- AI calls: provider latency and variability.

## 6.5 Mechanisms (conceptual)

- Proxy media for editing; originals only for final render.
- Windowed loading and eviction of frame data.
- Off-main-thread work for anything heavy.
- Hardware acceleration where the browser exposes it.
- Pre-generated thumbnails and waveforms.
- CDN and multi-layer caching with immutable artifact keys.
- Lazy loading of off-screen timeline content.
- Background rendering with progress events.

## 6.6 Capacity and scaling

Scaling dimensions: concurrent editors, media volume, render jobs, AI inference, storage, CDN traffic (Diagram 19). Workers scale on demand; the primary signal is queue age and depth. **Decision Point:** autoscaling policy and per-tenant fairness.

## 6.7 Measurement

Browser performance metrics (interaction latency, dropped frames, memory), pipeline metrics (stage durations, queue wait), render metrics (time per output minute, failure rate), agent metrics (latency, cost per run). Performance is checked against budgets on a repeatable test set of representative media.

## 6.8 Trade-offs to acknowledge

Quality vs latency (proxy fidelity), cost vs speed (worker pool size, eager analysis), consistency vs responsiveness (client vs server preview).

## 6.9 Open questions

Reference device class, supported media sizes/resolutions, proxy ladder, and acceptable degradation on weaker devices.

---

# 7. Reliability

## 7.1 Reliability goals

1. User work is not lost (strongest commitment).
2. Failures are visible, explained, and recoverable.
3. Partial failure degrades functionality rather than stopping everything.
4. Duplicate and retried operations are safe.

## 7.2 Failure domains and responses

| Failure | User-visible effect | System response |
| --- | --- | --- |
| Upload interruption | Upload paused | Resume from recorded state |
| Network loss in editor | Sync indicator offline | Queue mutations, resync on reconnect |
| Browser crash | Session lost | Restore from local persistence |
| Worker crash | Job stalls | Lease expiry, redelivery |
| Queue unavailable | Jobs not starting | Durable job record, re-enqueue when healthy |
| Render failure | Export fails | Classified retry, then clear failure message |
| AI provider failure | Agent unavailable or slow | Retry, fallback, or degrade to manual editing |
| Database failure | Writes blocked | Failover; read-only degraded mode |
| Object storage failure | Media unavailable | Retry with backoff, surface status |
| CDN failure | Slow or failed playback | Fall back to origin |

## 7.3 Idempotency boundaries

Upload parts, client mutations, ingest jobs, render jobs and their output keys, and agent tool calls each carry an identifier so repeating the operation produces the same outcome. **Decision Point:** who generates identifiers (client, API, or orchestrator) for each boundary.

## 7.4 Durability

- Structured data: backed up and replicated per policy.
- Media: object storage durability; **Decision Point:** replication and versioning policy, and whether originals have a second copy.
- Local state: bounded durability, treated as a convenience and recovery aid, not a source of truth.
- **Decision Point:** recovery point and recovery time objectives for each store.

## 7.5 Job and state-machine reliability

Long-running entities (uploads, assets, renders, agent runs) have explicit state machines with durable transitions, timeouts, and a path to terminal states. Stuck-state detection exists for each (for example a run that has not progressed). **Decision Point:** reconciliation process cadence and ownership.

## 7.6 Degradation modes

| Condition | Degraded behaviour |
| --- | --- |
| Agent unavailable | Manual editing fully available |
| Analysis unavailable | Agent restricted to project structure tools |
| Render backlog | Queue position and estimate shown |
| Sync unavailable | Local editing continues with pending indicator |
| Proxy not ready | Asset visible, not yet usable |

## 7.7 Conflict and data integrity

Sync conflicts follow the conflict-resolution Decision Point (docs 2 and 3). Integrity checks apply to uploads and to render outputs before they are reported as complete.

## 7.8 Operations and detection

Health indicators per component, alerting on queue age, error rates by class, stuck states, provider error rates, and storage errors. Correlation identifiers connect user actions to jobs, runs, and traces (Diagram 17).

## 7.9 Verification

**Proposal:** failure-injection scenarios for each row of 7.2 (kill a worker mid-render, drop the network mid-upload, fail the provider, simulate duplicate delivery), with defined expected outcomes. Interview-ready evidence is a documented scenario and observed result for each.

## 7.10 Open questions

Availability targets per tier (editor vs render vs agent), data retention and deletion guarantees, disaster recovery scope, and communication of failures to users (status surfaces, notifications).