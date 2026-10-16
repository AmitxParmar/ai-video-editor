# AI Video Editor + Cloud Media Pipeline: System Design Diagram Set

**Legend (all diagrams)**

- Solid arrow `-->` = synchronous request/response. Dashed arrow `-.->` = asynchronous / event / queue.
- Colour coding: blue = browser-side, green = backend services, amber = workers, purple = storage/data, pink = AI, grey = external systems.
- "Decision Point" = an architectural choice intentionally left open.
- C4 is drawn with Mermaid flowcharts and subgraphs (more reliable than Mermaid's experimental C4 syntax).

---

## 1. C4 System Context

**Purpose:** Who and what the system interacts with, at the outermost boundary.

```mermaid
flowchart LR
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111
  classDef be fill:#dcfce7,stroke:#16a34a,color:#111
  classDef wk fill:#fef3c7,stroke:#d97706,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111
  classDef ai fill:#fce7f3,stroke:#db2777,color:#111
  classDef ex fill:#e5e7eb,stroke:#6b7280,color:#111

  U["User - editor / creator"]:::ex
  ED["Browser Video Editor"]:::br
  BE["Backend / API"]:::be
  AG["AI Agent"]:::ai
  MP["Media Processing Platform"]:::wk
  OS[("Object Storage")]:::st
  CDN["CDN"]:::ex
  AUTH["Authentication Provider"]:::ex
  PROV["External AI Model Providers"]:::ex
  OBS["Monitoring / Observability"]:::ex

  U -->|"edits, instructs"| ED
  ED -->|"API calls"| BE
  ED -->|"sign-in"| AUTH
  BE -->|"verify identity"| AUTH
  ED -->|"direct chunk upload"| OS
  ED -->|"stream proxies / outputs"| CDN
  CDN -->|"origin fetch"| OS
  BE -->|"delegates instructions"| AG
  AG -->|"inference calls"| PROV
  AG -->|"tool calls"| BE
  BE -.->|"enqueue media/render jobs"| MP
  MP -->|"read / write media"| OS
  MP -.->|"status events"| BE
  BE -.->|"telemetry"| OBS
  AG -.->|"traces"| OBS
  MP -.->|"metrics"| OBS
  ED -.->|"browser metrics"| OBS
```

**Key entities:** User, Browser Editor, Backend/API, AI Agent, Media Processing Platform, Object Storage, CDN, Auth provider, AI providers, Observability.

**Decision Points**

- Is the AI Agent a separate deployable boundary or a module inside the Backend?
- Build vs buy for authentication (managed IdP vs self-hosted).
- Does the Media Processing Platform include the render workers, or are rendering and ingest separate platforms?

---

## 2. C4 Container Diagram

**Purpose:** The deployable containers and the main interactions between them.

```mermaid
flowchart TB
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111
  classDef be fill:#dcfce7,stroke:#16a34a,color:#111
  classDef wk fill:#fef3c7,stroke:#d97706,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111
  classDef ai fill:#fce7f3,stroke:#db2777,color:#111
  classDef ex fill:#e5e7eb,stroke:#6b7280,color:#111

  subgraph Client["Browser"]
    WE["Web Editor"]:::br
  end
  subgraph Edge["Edge"]
    CDN["CDN"]:::ex
  end
  subgraph Core["Backend Services"]
    API["API Service - gateway, authn/z"]:::be
    PTS["Project / Timeline Service"]:::be
    MS["Media Service - asset registry"]:::be
    US["Upload Service"]:::be
    RO["Render Orchestrator"]:::be
    MAS["Metadata / Analysis Service"]:::be
  end
  subgraph AIB["AI Boundary"]
    AGS["AI Agent Service"]:::ai
    ATL["Agent Tool Layer"]:::ai
  end
  subgraph Work["Workers"]
    RW["Render Workers"]:::wk
    IW["Ingest / Analysis Workers"]:::wk
  end
  subgraph Data["Data Stores"]
    Q[["Job Queue"]]:::st
    PG[("PostgreSQL")]:::st
    RD[("Redis")]:::st
    OS[("Object Storage")]:::st
  end
  PROV["External AI Providers"]:::ex
  OBS["Observability Stack"]:::ex

  WE -->|"REST / commands"| API
  WE <-.->|"WebSocket / SSE events"| API
  WE -->|"chunk upload via signed URLs"| OS
  WE -->|"proxy / output playback"| CDN
  CDN --> OS
  API --> PTS
  API --> MS
  API --> US
  API --> AGS
  API --> RO
  AGS --> ATL
  ATL --> PTS
  ATL --> MS
  ATL --> MAS
  ATL --> RO
  AGS -->|"inference"| PROV
  US -->|"multipart control"| OS
  US -.->|"upload complete"| Q
  MS -.->|"ingest jobs"| Q
  RO -.->|"render jobs"| Q
  Q -.-> IW
  Q -.-> RW
  IW --> OS
  RW --> OS
  IW -.->|"results"| MAS
  RW -.->|"progress"| RO
  PTS --> PG
  MS --> PG
  MAS --> PG
  RO --> PG
  AGS --> PG
  API --> RD
  AGS --> RD
  Core -.-> OBS
  AIB -.-> OBS
  Work -.-> OBS
```

**Key components:** as listed in the diagram. Persistent truth lives in PostgreSQL and Object Storage; Redis holds ephemeral state; the queue carries all long-running work.

**Decision Points**

- Modular monolith vs separate services for the six core backend boxes (the brief warns against unnecessary microservices).
- Which store owns job state: queue, PostgreSQL, or both.
- Whether Metadata/Analysis is a service or only a worker output schema.

---

## 3. Browser Editor Architecture

**Purpose:** Internal structure of the web editor and what is main-thread vs worker.

```mermaid
flowchart TB
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111
  classDef ex fill:#e5e7eb,stroke:#6b7280,color:#111

  subgraph UI["Main Thread - UI"]
    TL["Timeline UI"]:::br
    PP["Preview Player"]:::br
    MA["Media Assets Panel"]:::br
    AP["Agent Panel / Approval UI"]:::br
  end
  subgraph State["State Layer"]
    ES["Editor State - project state, selection, playhead"]:::br
    CMD["Command / Mutation Log"]:::br
  end
  subgraph Workers["Web Workers"]
    DW["Decode / Frame Worker"]:::br
    TW["Thumbnail / Waveform Worker"]:::br
    UW["Upload Worker"]:::br
  end
  subgraph Browser["Browser Media APIs"]
    WC["WebCodecs / Media APIs"]:::ex
    AUD["Web Audio"]:::ex
    GPU["Canvas / GPU rendering"]:::ex
  end
  LP[("Local Persistence - IndexedDB")]:::st
  NS["Network Sync Layer"]:::br
  SRV["Backend API / Events"]:::ex

  TL --> ES
  MA --> ES
  AP --> ES
  ES --> CMD
  ES --> PP
  PP --> GPU
  PP --> AUD
  PP -->|"frame requests"| DW
  DW --> WC
  MA -->|"derive previews"| TW
  MA -->|"file chunks"| UW
  CMD --> LP
  ES -->|"snapshot"| LP
  CMD -.->|"pending mutations"| NS
  NS <-.->|"sync / events"| SRV
  UW -->|"signed URL upload"| SRV
```

**Key components:** Timeline, Preview Player, Media Assets, Editor State, Local Persistence, Web Workers, Browser Media APIs, rendering/preview layer, Network Sync, project state.

**Decision Points**

- State model: command log vs snapshot vs both.
- Which preview path is canonical: browser decode of proxies, server-rendered preview, or hybrid.
- Where the main-thread / worker boundary sits for state (shared vs message-passed).

---

## 4. Video Processing Pipeline

**Purpose:** Lifecycle of a media asset from upload to final output.

```mermaid
flowchart LR
  classDef be fill:#dcfce7,stroke:#16a34a,color:#111
  classDef wk fill:#fef3c7,stroke:#d97706,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111
  classDef ex fill:#e5e7eb,stroke:#6b7280,color:#111

  UP["Upload"]:::br -.-> VAL["Media validation"]:::wk
  VAL -->|"rejected"| REJ["Rejected / quarantined"]:::be
  VAL -.-> META["Metadata extraction"]:::wk
  META -.-> TC["Transcoding"]:::wk
  TC -.-> PX["Proxy generation"]:::wk
  TC -.-> TH["Thumbnail generation"]:::wk
  TC -.-> WF["Waveform generation"]:::wk
  PX --> AS[("Asset storage")]:::st
  TH --> AS
  WF --> AS
  META --> DB[("Metadata in PostgreSQL")]:::st
  AS --> CDN["CDN delivery"]:::ex
  CDN --> TU["Timeline usage"]:::br
  TU -.-> FR["Final render"]:::wk
  AS -->|"original / mezzanine inputs"| FR
  FR --> OUT[("Output storage")]:::st
  OUT --> CDN
```

**Key components:** Validation, metadata, transcode, derived artifacts (proxy, thumbs, waveform), asset storage, CDN, final render, output storage.

**Decision Points**

- Are derived-artifact steps a fan-out of independent jobs or one sequential pipeline.
- Whether original media is retained untouched alongside a mezzanine format.
- Where transcript/scene analysis for the agent sits in this pipeline (always-on vs on demand).

---

## 5. Large File Upload Flow

**Purpose:** Resumable chunked upload directly to object storage with server-tracked state.

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant U as Upload API
  participant S as Upload State Store
  participant O as Object Storage
  participant Q as Event Queue

  B->>U: Initiate upload (file descriptor)
  U->>O: Create multipart session
  U->>S: Record upload session
  U-->>B: Session id + per-part signed URLs
  loop Each chunk
    B->>O: Upload part
    O-->>B: Part receipt
    B-->>U: Report part completed
    U->>S: Update part state
  end
  Note over B,S: Interrupted? Browser asks Upload API for session state, then resends only missing parts
  B->>U: Complete upload
  U->>O: Finalize multipart
  U->>S: Mark complete
  U-)Q: Publish UploadCompleted event
```

**Key components:** Browser upload logic, Upload API, multipart session, Upload State store, object storage, completion event.

**Decision Points**

- Upload state authority: PostgreSQL, Redis, or object storage's own part listing.
- Chunk size policy and parallelism (fixed vs adaptive).
- Integrity verification: per-part checksum vs whole-file.
- Session expiry and orphan cleanup policy.

---

## 6. Video Rendering Architecture

**Purpose:** From a project definition to a delivered render, including progress and failure paths.

```mermaid
flowchart TB
  classDef be fill:#dcfce7,stroke:#16a34a,color:#111
  classDef wk fill:#fef3c7,stroke:#d97706,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111

  TLD["Timeline / project definition (immutable snapshot)"]:::be
  RR["Render request"]:::be
  Q[["Render Queue"]]:::st
  RO["Render Orchestrator"]:::be
  RW["Render Worker"]:::wk
  MI[("Media inputs - Object Storage")]:::st
  ENG["FFmpeg / media engine"]:::wk
  IA[("Intermediate artifacts")]:::st
  FO[("Final output")]:::st
  PR["Progress reporting"]:::be
  FAIL["Failure / retry handling"]:::be
  DLQ[["Dead-letter / failed jobs"]]:::st
  RES["Result delivery - event + signed URL"]:::be
  CL["Browser"]:::br

  TLD --> RR
  RR -.-> Q
  Q -.-> RO
  RO -.->|"assign / split"| RW
  MI --> RW
  RW --> ENG
  ENG --> IA
  IA --> ENG
  ENG --> FO
  RW -.->|"heartbeat, % complete"| PR
  PR -.-> CL
  RW -.->|"error"| FAIL
  FAIL -.->|"retry"| Q
  FAIL -.->|"exhausted"| DLQ
  FO --> RES
  RES -.-> CL
```

**Key components:** Project snapshot, render request, queue, orchestrator, workers, media engine, intermediate artifacts, output, progress, retry/DLQ, delivery.

**Decision Points**

- Render granularity: whole job per worker vs segmented/parallel with stitch.
- Server-side render only, or also client-side export for small projects.
- Retry policy per failure class (transient vs deterministic).
- Retention of intermediate artifacts for resume.

---

## 7. AI Editing Agent Architecture

**Purpose:** How a natural-language instruction becomes validated, approved timeline changes.

```mermaid
flowchart TB
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111
  classDef be fill:#dcfce7,stroke:#16a34a,color:#111
  classDef ai fill:#fce7f3,stroke:#db2777,color:#111
  classDef ex fill:#e5e7eb,stroke:#6b7280,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111

  UI["User instruction"]:::br
  AG["Agent runtime"]:::ai
  PB{{"Planner / reasoning boundary"}}:::ai
  MPA["Model provider abstraction"]:::ai
  PROV["External model providers"]:::ex
  TOOLS["Tool layer"]:::ai
  subgraph Understanding["Video Understanding"]
    TR["Transcript"]:::be
    SD["Scene detection"]:::be
    FCA["Frame / clip analysis"]:::be
  end
  TM["Timeline manipulation tools"]:::be
  VAL["Validation"]:::be
  HA["Human approval"]:::br
  RR["Render request"]:::be
  PS[("Project state")]:::st

  UI --> AG
  AG --> PB
  PB <-->|"inference"| MPA
  MPA --> PROV
  PB -->|"tool intents"| TOOLS
  TOOLS --> TR
  TOOLS --> SD
  TOOLS --> FCA
  TOOLS --> TM
  TM --> PS
  TM --> VAL
  VAL -->|"fail: feedback"| PB
  VAL -->|"pass"| HA
  HA -->|"reject / revise"| PB
  HA -->|"approve"| RR
  Understanding -->|"results"| PB
```

**Key components:** Agent runtime, planner boundary, provider abstraction, tool layer, video understanding, timeline tools, validation, approval gate, render request.

**Decision Points**

- Where the planner lives (single LLM loop, multi-step planner/executor split, or multi-agent).
- Do agent edits apply to the live project, a branch/draft, or a proposal diff.
- Is human approval required for every change, only before render, or configurable.
- Whether video understanding is precomputed at ingest or lazily invoked by tools.

---

## 8. AI Agent Tool Architecture

**Purpose:** Catalogue of tools, grouped by side-effect class.

```mermaid
flowchart LR
  classDef ro fill:#dbeafe,stroke:#2563eb,color:#111
  classDef mu fill:#fef3c7,stroke:#d97706,color:#111
  classDef cost fill:#fce7f3,stroke:#db2777,color:#111
  classDef core fill:#dcfce7,stroke:#16a34a,color:#111

  AG["Agent"]:::core --> REG["Tool Registry / Dispatcher - schema, permissions, tracing"]:::core

  REG --> R1
  REG --> M1
  REG --> A1
  REG --> X1

  subgraph R["Read-only inspection"]
    R1["inspect_project"]:::ro
    R2["inspect_media"]:::ro
    R3["search_transcript"]:::ro
    R4["find_clip"]:::ro
  end
  subgraph A["Analysis - compute / AI cost"]
    A1["detect_scenes"]:::cost
    A2["generate_caption"]:::cost
  end
  subgraph M["Timeline mutation"]
    M1["trim_clip"]:::mu
    M2["split_clip"]:::mu
    M3["move_clip"]:::mu
    M4["add_text"]:::mu
    M5["add_audio"]:::mu
  end
  subgraph X["Preview / output"]
    X1["preview_changes"]:::ro
    X2["render_video"]:::cost
  end
  R1 --- R2 --- R3 --- R4
  M1 --- M2 --- M3 --- M4 --- M5
```

**Key components:** Registry/dispatcher, read-only tools, analysis tools, mutating tools, preview/render tools.

**Decision Points**

- Tool exposure protocol (MCP vs internal function-calling schema).
- Granularity of tools (many primitives vs fewer composite operations).
- Which tools need approval gating (all mutations, only `render_video`).
- Idempotency and undo semantics for each mutating tool.
- Whether `generate_caption` is an analysis tool or a mutation (writes text clips).

---

## 9. AI Agent State Machine

**Purpose:** Lifecycle states of a single agent run.

```mermaid
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PLANNING: instruction accepted
  PLANNING --> EXECUTING: plan ready
  PLANNING --> FAILED: provider error exhausted
  EXECUTING --> WAITING_FOR_TOOL: tool invoked
  WAITING_FOR_TOOL --> EXECUTING: tool result
  WAITING_FOR_TOOL --> FAILED: tool error exhausted
  EXECUTING --> PLANNING: replan needed
  EXECUTING --> VALIDATING: edits proposed
  VALIDATING --> EXECUTING: validation failed, revise
  VALIDATING --> WAITING_FOR_APPROVAL: validation passed
  WAITING_FOR_APPROVAL --> PLANNING: user requests changes
  WAITING_FOR_APPROVAL --> RENDERING: approved
  WAITING_FOR_APPROVAL --> COMPLETED: approved, no render needed
  WAITING_FOR_APPROVAL --> CANCELLED: rejected / timeout
  RENDERING --> COMPLETED: render succeeded
  RENDERING --> FAILED: render failed
  CREATED --> CANCELLED: user cancel
  PLANNING --> CANCELLED: user cancel
  EXECUTING --> CANCELLED: user cancel
  WAITING_FOR_TOOL --> CANCELLED: user cancel
  COMPLETED --> [*]
  FAILED --> [*]
  CANCELLED --> [*]
```

**Key states:** CREATED, PLANNING, EXECUTING, WAITING_FOR_TOOL, VALIDATING, WAITING_FOR_APPROVAL, RENDERING, COMPLETED, FAILED, CANCELLED.

**Decision Points**

- Is state persisted per transition (durable, resumable) or in memory only.
- Bounds on replan/revise loops (max iterations, budget).
- Whether FAILED is terminal or resumable.
- Approval timeout behaviour.
- Is RENDERING part of the agent run or a separate linked Render lifecycle.

---

## 10. Project / Timeline Data Model

**Purpose:** Core persistent entities and their relationships.

```mermaid
erDiagram
  USER ||--o{ PROJECT : owns
  PROJECT ||--o{ TIMELINE : has
  TIMELINE ||--o{ TRACK : contains
  TRACK ||--o{ CLIP : holds
  ASSET ||--o{ CLIP : "source of"
  PROJECT ||--o{ ASSET : references
  CLIP ||--o{ EFFECT : "has"
  CLIP ||--o{ TRANSITION : "joined by"
  CLIP ||--o| TEXT : "may be"
  CLIP ||--o| AUDIO : "may be"
  PROJECT ||--o{ RENDER : produces
  TIMELINE ||--o{ RENDER : "snapshot of"
  PROJECT ||--o{ AGENT_RUN : has
  AGENT_RUN ||--o{ TOOL_CALL : issues
  AGENT_RUN |o--o{ RENDER : requests
  USER ||--o{ ASSET : uploads

  USER { string id PK }
  PROJECT { string id PK }
  TIMELINE { string id PK }
  TRACK { string id PK }
  CLIP { string id PK }
  ASSET { string id PK }
  AUDIO { string id PK }
  TEXT { string id PK }
  EFFECT { string id PK }
  TRANSITION { string id PK }
  RENDER { string id PK }
  AGENT_RUN { string id PK }
  TOOL_CALL { string id PK }
```

**Key entities:** as listed. Attributes are omitted deliberately; only identities and relationships are in scope.

**Decision Points**

- Timeline as normalized rows vs a versioned document (JSON) per project.
- Text and Audio as separate entities vs clip subtypes.
- Version/revision history: per-edit, per-snapshot, or none.
- Whether Assets are user-scoped, project-scoped, or workspace-scoped (tenancy).

---

## 11. Real-Time Architecture

**Purpose:** Live channels between clients and the backend: state, presence, progress.

```mermaid
flowchart LR
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111
  classDef be fill:#dcfce7,stroke:#16a34a,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111
  classDef wk fill:#fef3c7,stroke:#d97706,color:#111

  C1["Editor client A"]:::br
  C2["Editor client B"]:::br
  GW["Realtime Gateway - WebSocket / SSE"]:::be
  CS{{"Collaboration / sync boundary"}}:::be
  PTS["Project / Timeline Service"]:::be
  PRES["Presence"]:::be
  RD[("Redis pub/sub")]:::st
  PG[("PostgreSQL")]:::st
  RW["Render Workers"]:::wk
  AGS["AI Agent Service"]:::be

  C1 <-->|"mutations / state updates"| GW
  C2 <-->|"mutations / state updates"| GW
  GW --> CS
  CS --> PTS
  PTS --> PG
  GW --> PRES
  PRES --> RD
  PTS -.->|"project change events"| RD
  RD -.->|"fan-out"| GW
  RW -.->|"render progress events"| RD
  AGS -.->|"agent status events"| RD
```

**Key components:** Clients, realtime gateway, sync boundary, presence, pub/sub fan-out, progress/render/agent events.

**Decision Points**

- WebSocket vs SSE (bidirectional vs server-push plus REST).
- Is multi-user collaborative editing in scope at all, or single-editor with multi-device.
- Conflict model if collaborative (see Diagram 12).
- Whether agent edits appear to collaborators live or only after approval.

---

## 12. Offline-First / Local-First Architecture

**Purpose:** What lives locally, how changes sync, and where conflicts can occur.

```mermaid
flowchart TB
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111
  classDef be fill:#dcfce7,stroke:#16a34a,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111

  subgraph Local["Browser"]
    LS["Local editor state"]:::br
    IDB[("IndexedDB")]:::st
    LM["Local media metadata / cached proxies"]:::br
    PM["Pending mutations queue"]:::br
    SL["Sync layer"]:::br
  end
  subgraph Remote["Server"]
    SS["Server state - PostgreSQL"]:::st
    API["Sync API"]:::be
  end
  CB{{"Conflict boundary"}}:::be
  REC["Recovery after reconnect"]:::br

  LS --> IDB
  LM --> IDB
  LS -->|"user edit"| PM
  PM --> IDB
  PM -.->|"flush when online"| SL
  SL <-.->|"push / pull"| API
  API --> CB
  CB --> SS
  CB -.->|"conflict result"| SL
  SL --> REC
  REC -->|"replay or rebase pending"| LS
  IDB -->|"restore after crash"| LS
```

**Key components:** Local state, IndexedDB, local media metadata, pending mutations, sync layer, server state, conflict boundary, reconnect recovery.

**Decision Points**

- Conflict strategy: last-writer-wins, per-field merge, operational transform, or CRDT.
- Whether offline is full local-first (local is source of truth) or cache-with-queue (server is truth).
- Local media policy: store originals locally, proxies only, or references only.
- Storage quota and eviction behaviour.

---

## 13. Performance Architecture

**Purpose:** Where performance pressure arises and which mechanisms address it.

```mermaid
flowchart LR
  classDef pr fill:#fee2e2,stroke:#dc2626,color:#111
  classDef mc fill:#dcfce7,stroke:#16a34a,color:#111

  P1["Large video files"]:::pr --> M1["Proxy media for editing"]:::mc
  P1 --> M2["Chunked / range access"]:::mc
  P2["Browser memory limits"]:::pr --> M3["Windowed frame buffers"]:::mc
  P2 --> M4["Lazy loading of off-screen clips"]:::mc
  P3["Main-thread jank"]:::pr --> M5["Web Workers for decode / analysis"]:::mc
  P4["Preview smoothness"]:::pr --> M6["GPU / browser acceleration"]:::mc
  P4 --> M1
  P5["Network latency and egress"]:::pr --> M7["CDN delivery"]:::mc
  P5 --> M8["Caching - browser, CDN, Redis"]:::mc
  P6["Timeline scrubbing cost"]:::pr --> M9["Pre-generated thumbnails / waveforms"]:::mc
  P7["Long exports"]:::pr --> M10["Background rendering on workers"]:::mc
```

**Key components:** Problem to mechanism mapping across client, network, and backend.

**Decision Points**

- Proxy resolution/codec ladder (single proxy vs multiple renditions).
- Streaming format for preview (progressive file vs segmented streaming).
- Memory budget policy and degradation behaviour on low-end devices.
- Cache invalidation rules when source assets are replaced.

---

## 14. Reliability / Failure Diagram

**Purpose:** Failure domains and the recovery path attached to each.

```mermaid
flowchart TB
  classDef f fill:#fee2e2,stroke:#dc2626,color:#111
  classDef r fill:#dcfce7,stroke:#16a34a,color:#111
  classDef i fill:#fef3c7,stroke:#d97706,color:#111

  F1["Upload failure"]:::f --> R1["Resume from persisted part state"]:::r
  F2["Network interruption"]:::f --> R2["Queue pending mutations, reconnect, resync"]:::r
  F3["Browser crash"]:::f --> R3["Restore from IndexedDB"]:::r
  F4["Worker crash"]:::f --> R4["Lease timeout, job redelivered"]:::r
  F5["Queue failure"]:::f --> R5["Producers retry; durable job record re-enqueues"]:::r
  F6["Render failure"]:::f --> R6["Retry policy, then dead-letter, user notified"]:::r
  F7["AI provider failure"]:::f --> R7["Retry / fallback provider / degrade to manual editing"]:::r
  F8["Database failure"]:::f --> R8["Failover replica, read-only degraded mode"]:::r
  F9["Object-storage failure"]:::f --> R9["Retry with backoff, surface error, no data loss claims"]:::r
  F10["CDN failure"]:::f --> R10["Origin fallback"]:::r

  I1{{"Idempotency: upload part ids"}}:::i -.- R1
  I2{{"Idempotency: mutation ids"}}:::i -.- R2
  I3{{"Idempotency: job id / output key"}}:::i -.- R4
  I4{{"Idempotency: tool-call id"}}:::i -.- R7
```

**Key components:** Ten failure domains, their recovery paths, and four idempotency boundaries (upload parts, client mutations, render jobs, agent tool calls).

**Decision Points**

- Delivery semantics for the queue (at-least-once with idempotent consumers vs stronger).
- RPO/RTO targets for PostgreSQL and object storage.
- Multi-provider fallback policy for AI (cost, quality parity).
- Whether failed agent runs are resumable or restarted.

---

## 15. End-to-End Request Flow

**Purpose:** One complete journey from instruction to playable result.

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant B as Browser
  participant A as API
  participant G as AI Agent
  participant M as Media Analysis
  participant T as Tool Layer
  participant P as Project/Timeline Svc
  participant V as Validator
  participant Q as Queue
  participant W as Render Worker
  participant O as Object Storage
  participant C as CDN

  U->>B: Instruction
  B->>A: Submit instruction (project id)
  A->>G: Create agent run
  G->>M: Request analysis (transcript, scenes)
  M-->>G: Analysis results
  G->>T: Tool calls
  T->>P: Timeline mutation (proposed)
  P-->>T: Result
  T->>V: Validate changes
  V-->>G: Valid / issues
  G-->>B: Proposed edit for approval (event)
  U->>B: Approve
  B->>A: Approve and render
  A-)Q: Enqueue render job
  Q-)W: Deliver job
  W->>O: Read inputs, write output
  W-)A: Completion event
  A-)B: Render ready event
  B->>C: Fetch output
  C->>O: Origin fetch on miss
  C-->>B: Video stream
```

**Key components:** All hops from user to CDN; sync request/response mixed with async events (`-)`).

**Decision Points**

- Does the browser talk to the Agent via API (proxied) or directly over a realtime channel.
- Is analysis run on demand inside the loop or read from precomputed results.
- Where validation sits: inside the tool layer, inside Project Service, or both.

---

## 16. Security Architecture

**Purpose:** Trust boundaries, access control points, and sensitive data handling.

```mermaid
flowchart TB
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111
  classDef be fill:#dcfce7,stroke:#16a34a,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111
  classDef ai fill:#fce7f3,stroke:#db2777,color:#111
  classDef ex fill:#e5e7eb,stroke:#6b7280,color:#111

  B["Browser (untrusted)"]:::br
  IDP["Authentication Provider"]:::ex
  subgraph TB1["Trusted backend boundary"]
    GW["API gateway - authn"]:::be
    AZ["Authorization - project ownership checks"]:::be
    TI["Tenant isolation"]:::be
    SIGN["Signed URL issuer"]:::be
    AUD["Audit event log"]:::be
    SEC[("Secrets manager")]:::st
  end
  OS[("Object Storage - private")]:::st
  CDN["CDN - signed access"]:::ex
  AGS["AI Agent Service"]:::ai
  PROV["External AI providers"]:::ex
  SD{{"Sensitive data boundary: user media and transcripts leaving to providers"}}:::ai

  B -->|"credentials"| IDP
  IDP -->|"token"| B
  B -->|"token"| GW
  GW --> AZ --> TI
  AZ --> SIGN
  SIGN -->|"short-lived URLs"| B
  B --> OS
  B --> CDN
  CDN --> OS
  AGS -->|"fetches provider key"| SEC
  AGS --> SD --> PROV
  GW -.-> AUD
  AZ -.-> AUD
  AGS -.-> AUD
  SIGN -.-> AUD
```

**Key components:** AuthN, AuthZ, ownership, signed URLs, secrets, model credentials, tenant isolation, audit, sensitive-data boundary.

**Decision Points**

- Tenant isolation model: shared schema with row-level scoping, schema-per-tenant, or separate stacks.
- Agent tool authorization: runs with the user's permissions or a scoped service identity.
- Provider data policy: what media/transcript content may be sent externally and retention terms.
- Signed URL lifetime and CDN token strategy.

---

## 17. Observability Architecture

**Purpose:** Telemetry sources, correlation, and the signals they feed.

```mermaid
flowchart LR
  classDef src fill:#dbeafe,stroke:#2563eb,color:#111
  classDef pipe fill:#fef3c7,stroke:#d97706,color:#111
  classDef out fill:#dcfce7,stroke:#16a34a,color:#111

  subgraph S["Telemetry sources"]
    BR["Browser performance metrics"]:::src
    BE["Backend services"]:::src
    AG["Agent traces"]:::src
    TC["Tool-call traces"]:::src
    RJ["Render job metrics"]:::src
    QM["Queue metrics"]:::src
  end
  CID{{"Correlation ID: request / project / agent-run / job"}}:::pipe
  COL["Telemetry collection pipeline"]:::pipe
  subgraph O["Signal stores and views"]
    LG["Logs"]:::out
    MT["Metrics"]:::out
    TR["Distributed traces"]:::out
    ER["Error tracking"]:::out
  end
  AL["Alerting / dashboards"]:::out

  BR -.-> CID
  BE -.-> CID
  AG -.-> CID
  TC -.-> CID
  RJ -.-> CID
  QM -.-> CID
  CID -.-> COL
  COL -.-> LG
  COL -.-> MT
  COL -.-> TR
  COL -.-> ER
  LG --> AL
  MT --> AL
  TR --> AL
  ER --> AL
```

**Key components:** Six source classes, correlation identifiers, collection pipeline, four signal types, alerting.

**Decision Points**

- Trace propagation across async boundaries (queue to worker, agent to tool).
- Storage/retention of agent traces that may contain user content (links to Diagram 16).
- Vendor-neutral instrumentation vs a single vendor stack.
- Sampling policy for browser metrics and traces.

---

## 18. Deployment Architecture

**Purpose:** Production infrastructure layout and network zones.

```mermaid
flowchart TB
  classDef br fill:#dbeafe,stroke:#2563eb,color:#111
  classDef be fill:#dcfce7,stroke:#16a34a,color:#111
  classDef wk fill:#fef3c7,stroke:#d97706,color:#111
  classDef st fill:#f3e8ff,stroke:#9333ea,color:#111
  classDef ex fill:#e5e7eb,stroke:#6b7280,color:#111

  B["Browser"]:::br
  subgraph Edge["Edge"]
    CDN["CDN + static editor assets"]:::ex
    LB["Load balancer / WAF"]:::ex
  end
  subgraph App["Application zone"]
    API["API services - autoscaled"]:::be
    RT["Realtime gateway"]:::be
    AGS["AI Agent Service"]:::be
  end
  subgraph Wk["Worker zone"]
    IW["Ingest / analysis workers"]:::wk
    RW["Render worker pool"]:::wk
  end
  subgraph Data["Data zone"]
    Q[["Queue"]]:::st
    PG[("PostgreSQL - primary + replica")]:::st
    RD[("Redis")]:::st
    OS[("Object Storage")]:::st
  end
  PROV["AI providers"]:::ex
  MON["Monitoring"]:::ex

  B --> CDN
  B --> LB
  LB --> API
  LB --> RT
  API --> AGS
  AGS -->|"egress"| PROV
  API --> PG
  API --> RD
  RT --> RD
  API -.-> Q
  Q -.-> IW
  Q -.-> RW
  IW --> OS
  RW --> OS
  CDN --> OS
  B -->|"signed URL upload"| OS
  App -.-> MON
  Wk -.-> MON
```

**Key components:** Edge, application zone, worker zone, data zone, external providers, monitoring.

**Decision Points**

- Single cloud vs multi-cloud; managed services vs self-run for queue, DB, Redis.
- Container orchestration vs serverless for each tier (workers especially).
- GPU vs CPU render pools.
- Region strategy and data residency.

---

## 19. Cost / Scaling Architecture

**Purpose:** Scaling dimensions, what each drives, and the cost centre it hits.

```mermaid
flowchart LR
  classDef dim fill:#dbeafe,stroke:#2563eb,color:#111
  classDef comp fill:#dcfce7,stroke:#16a34a,color:#111
  classDef cost fill:#fee2e2,stroke:#dc2626,color:#111

  D1["Concurrent editors"]:::dim --> C1["Realtime gateway + API"]:::comp --> K1["Compute / connections"]:::cost
  D2["Uploaded media volume"]:::dim --> C2["Ingest workers + storage"]:::comp --> K2["Storage + transcode compute"]:::cost
  D3["Render jobs"]:::dim --> C3["Queue depth to worker pool"]:::comp --> K3["Worker compute - dominant cost"]:::cost
  D4["AI inference"]:::dim --> C4["Agent service + provider calls"]:::comp --> K4["Per-token / per-frame provider cost"]:::cost
  D5["Storage"]:::dim --> C5["Object storage tiers"]:::comp --> K5["Retention of originals, proxies, renders"]:::cost
  D6["CDN traffic"]:::dim --> C6["CDN"]:::comp --> K6["Egress"]:::cost
  D3 --> D7["Queue depth"]:::dim
  D7 -.->|"drives autoscaling"| C3
```

**Key components:** Seven scaling dimensions mapped to the component that absorbs them and the cost they incur.

**Decision Points**

- Autoscaling signal for workers (queue depth, age of oldest job, or both).
- Per-user quotas and fairness across tenants for renders and AI calls.
- Storage lifecycle policy for proxies and old renders.
- Use of spot/preemptible capacity for workers (ties to Diagram 14 retry design).

---

## 20. Technology-to-Component Mapping

**Purpose:** Candidate technologies per responsibility. Entries are candidates, not selections.

```mermaid
flowchart LR
  classDef resp fill:#dcfce7,stroke:#16a34a,color:#111
  classDef tech fill:#e5e7eb,stroke:#6b7280,color:#111

  R1["Editor UI and routing"]:::resp --- T1["React / Next.js + TypeScript"]:::tech
  R2["Frame decode and encode in browser"]:::resp --- T2["WebCodecs"]:::tech
  R3["Off-main-thread compute"]:::resp --- T3["Web Workers"]:::tech
  R4["Local persistence"]:::resp --- T4["IndexedDB"]:::tech
  R5["API and orchestration services"]:::resp --- T5["FastAPI or Node.js - Decision Point"]:::tech
  R6["System of record"]:::resp --- T6["PostgreSQL"]:::tech
  R7["Ephemeral state, pub/sub, cache"]:::resp --- T7["Redis"]:::tech
  R8["Async job transport"]:::resp --- T8["Queue system - Decision Point"]:::tech
  R9["Media and render storage"]:::resp --- T9["S3-compatible object storage"]:::tech
  R10["Transcode and render engine"]:::resp --- T10["FFmpeg"]:::tech
  R11["Reasoning, transcription, vision"]:::resp --- T11["External AI model providers"]:::tech
  R12["Agent tool exposure"]:::resp --- T12["MCP or internal tool schema - Decision Point"]:::tech
  R13["Telemetry"]:::resp --- T13["Observability stack - Decision Point"]:::tech
```

**Key components:** Thirteen responsibilities paired with candidate technologies.

**Decision Points**

- Backend language/runtime (FastAPI vs Node.js) and whether workers share it.
- Queue technology and its delivery guarantees.
- Provider selection and abstraction depth.
- MCP vs internal tool protocol.
- Observability vendor vs self-hosted open stack.

---

## Cross-Cutting Decision Points

1. Collaboration scope: single-editor vs multi-user (affects Diagrams 11, 12, 10).
2. Timeline representation: document vs relational (affects Diagrams 10, 12, 7).
3. Agent autonomy: how much can be applied without human approval (affects 7, 8, 9).
4. Preview strategy: client-side vs server-side vs hybrid (affects 3, 13, 6).
5. Tenancy and data-sharing with external AI providers (affects 16, 17, 7).