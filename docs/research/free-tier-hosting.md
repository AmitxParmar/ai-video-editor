# Free-Tier Hosting Landscape (2026)

**Wayfinder research ticket:** [#53](https://github.com/AmitxParmar/ai-video-editor/issues/53) — research the 2026 free-tier hosting landscape for the video-editor stack: a Rust API container, FFmpeg CPU workers, managed Postgres, Redis/queue, S3-compatible storage, and CDN egress.

**Researched:** 2026-10-18. **Method:** primary sources only — provider pricing pages and official docs, each fetched on 2026-10-18 and linked in [Sources](#sources). Where a number is an inference (unit conversion, workload math) it is marked. Free tiers change constantly; re-verify before committing money.

**Assumed workload:** small demo scale. Uploads of short clips, proxy generation, occasional FFmpeg renders, AI API calls. Idle cost must be ~$0/month; bursts are fine. The API is a Rust container; workers run FFmpeg.

---

## 0. TL;DR

- **The single biggest cost trap for video is egress, and the market has reshuffled around it.** Cloudflare R2 charges $0 egress at any volume; Backblaze B2 gives free egress up to 3× average monthly storage (and unlimited via Cloudflare/Fastly/bunny.net partners); Modal includes 1 TiB/month of egress in its Starter plan; Oracle throws in a stunning 10 TB/month of outbound data on Always Free. Everyone else meters it (Render $0.15/GB, Northflank $0.06/GB, Railway $0.05/GB, Fly $0.02–$0.12/GB).
- **There is no longer a meaningful "free VPS forever" tier except Oracle Always Free** — and Oracle's free ARM capacity is hard to get and *can be reclaimed for being idle* (95th-percentile CPU, network, and memory all under 20% over 7 days).
- **Free tiers of "free PaaS" are mostly sleeping demos now.** Render free web services sleep after 15 min (~1 min cold start) and its free Postgres *expires after 30 days*. Fly.io has no free tier for new orgs, only a 2-hour/7-day trial. Railway's free plan gives $1/month of usage credits — roughly 29 hours/month of a 1 vCPU / 0.5 GB service, not always-on. Koyeb's only free compute is a 5-hour Postgres trial.
- **The strongest $0 stack in 2026 is assembled from parts, not one provider:** Render free (API) + Modal Starter credits (FFmpeg) + Neon Free (Postgres) + Upstash QStash/Redis (queue) + Cloudflare R2 or Backblaze B2 (storage). All five have permanent free tiers; only R2 may ask for a payment method at activation.
- **Neon is the best free-forever Postgres for a demo:** permanent free plan, no credit card, 100 CU-hours/project, scale-to-zero after 5 min, and a "claimable" instant project with no signup at all. Supabase Free is fine until it *pauses your project after ~7 days of inactivity*; Render Free Postgres is disqualified by its 30-day expiry.
- **For async job transport:** Upstash QStash (1,000 msgs/day, HTTP push, retries, no card) is the cheapest path; CloudAMQP's free shared RabbitMQ (1M msgs/month, 20 connections) is the real-AMQP option; Upstash Redis free (250 MB, 500k commands/month) covers Redis-streams demos; AWS SQS (1M requests/month) and GCP Pub/Sub (10 GiB/month) are the hyperscaler options but need full cloud accounts with cards.

---

## 1. How to read the tables

Every provider table lists **what's actually free**, the **caveats that will bite a video pipeline** (sleeping, expiries, quotas, egress, idle reclamation), and **signup reality** (card required? region capacity? trial-only?). "Permanent" means the free tier is not time-limited; "trial" means it expires.

---

## 2. The egress trap (read this first)

Video is egress-heavy: every preview, proxy, and final render that a user watches crosses the network. 2026 egress economics on free/relevant tiers:

| Provider | Egress policy | Meaning for a demo |
| --- | --- | --- |
| Cloudflare R2 | **$0 egress, always** (any volume) | The default choice for serving media |
| Backblaze B2 | Free up to 3× average monthly storage; $0.01/GB beyond; **unlimited free via CDN partners** (Cloudflare, Fastly, bunny.net, …) | Serve through Cloudflare and egress is $0 |
| Modal | 1 TiB/month free on Starter | Generous for demo video traffic |
| Oracle Always Free | **10 TB/month** outbound | Effectively unlimited for a demo |
| Google Cloud Storage | 100 GB/month free egress from North America (excl. China/Australia) | Fine for a demo; watch GCE/Cloud Run's 1 GB limits |
| Fly.io | $0.02/GB (NA/EU), $0.04/GB (APAC), $0.12/GB (Africa/India) | Cheap but not free |
| Railway | $0.05/GB | Object storage is $0.015/GB-month with free egress |
| Northflank | $0.06/GB | Expensive for video |
| Render (Hobby) | 5 GB/month included, then $0.15/GB; **no payment method → free services suspended** on overage | Keep serving off Render |
| Koyeb | 1 TB/month included on Pro; then $0.02/GB (US/EU), $0.04/GB (Asia) | Not usable at $0 idle (Pro is $29/mo) |

**Practical rule:** store on R2 (or B2+Cloudflare), render on Modal, and never serve user-facing media from a compute platform that meters egress.

---

## 3. Compute hosts (API / containers)

| Provider | What's free | Caveats | Signup reality |
| --- | --- | --- | --- |
| **Render** (Hobby) | Free web services: 0.1 CPU / 512 MB, Docker supported, custom domains, TLS; 750 free instance hours/workspace/month; free static sites | **Sleeps after 15 min idle** (~1 min cold start); ephemeral filesystem; no persistent disks; no SSH; no scaling; can't receive private traffic; **may suspend services that initiate "uncommonly high" outbound traffic (e.g., pulling from object storage)**; 5 GB/month bandwidth on Hobby; free **Postgres expires 30 days after creation** (14-day grace, then deleted); one free Postgres and one free Key Value (25 MB, in-memory) per workspace | Easy: email/GitHub, no card needed for free tier (without a card, overage = suspension instead of billing) |
| **Fly.io** | Free trial: up to 2 h of Machine runtime or 7 days, no card; ongoing free items: first 10 GB/month of volume snapshots, first 10 single-hostname SSL certs, inbound transfer, shared IPv4/IPv6 | **No free tier for new orgs**; after trial a card is required. Smallest always-on Machine (shared-cpu-1x 256 MB) = **$2.19/month**; 512 MB = $3.69; volumes $0.15/GB/mo; auto-stop/auto-start supported (stopped Machine costs only rootfs, $0.15/GB/mo); Managed Postgres starts at **$38/month** | Card only after the trial; self-serve |
| **Railway** | **Free Trial:** $5 one-time credit, 30 days, **no card required**, up to 2 vCPU/1 GB per service. **Free plan:** $1/month of usage credits, 1 project, 3 services, 1 vCPU/0.5 GB per service, 1 replica, 500 MB max volume, no custom domains | $1/month ≈ **~29 hours** of a 1 vCPU / 0.5 GB service (inference: $20/vCPU/mo + $10/GB/mo rates) — not always-on; egress $0.05/GB; stopped services cost nothing | Easiest free start (explicitly no card for trial) |
| **Koyeb** | No free compute tier; **free Postgres: 5 hours** (0.25 vCPU/1 GB); Pro plan $29/mo + compute with $10 included compute and 1 TB bandwidth | Koyeb is [joining Mistral AI](https://www.koyeb.com/blog/koyeb-is-joining-mistral-ai-to-build-the-future-of-ai-infrastructure) — roadmap uncertainty; scale-to-zero exists but is a paid feature | Self-serve; card for Pro |
| **Northflank** | **Sandbox tier:** 2 free services, 1 free database, 2 free cron jobs, "always-on compute — no sleeping" | Egress $0.06/GB is the highest here; disk $0.15/GB/mo; free tier specs aren't fully published — treat as "2 small always-on containers + 1 DB"; paid plans are consumption-based ($0.01667/vCPU/hr + $0.00833/GB/hr) | Self-serve signup |
| **Oracle Cloud (Always Free)** | Up to **2 OCPU / 12 GB ARM** (A1.Flex; 1,500 OCPU-h + 9,000 GB-h per month), 2× AMD micro VMs (1/8 OCPU, 1 GB); 200 GB block volume; **10 TB/month egress**; 2 Autonomous DBs (1 OCPU, 20 GB each); MySQL HeatWave 50 GB; 1 flexible LB (10 Mbps) | **Idle reclamation:** instances can be reclaimed if 95th-pct CPU <20%, network <20%, and memory <20% over 7 days — a sleeping demo can literally be deleted; **"out of host capacity" is common for A1 in popular regions**; signup requires card verification (small temporary hold, commonly $1) and is the most failure-prone of all options; no managed Postgres/Redis, you run everything yourself | Hardest: card verification, home-region choice is permanent, capacity lottery |
| **Google Cloud (Always Free)** | **Cloud Run:** 2M requests, 180k vCPU-seconds, 360k GiB-seconds, 1 GB NA egress/month, scale-to-zero. **Compute Engine:** one non-preemptible `e2-micro` (us-west1/us-central1/us-east1), 30 GB standard disk, 1 GB NA egress/month. Plus GKE Autopilot control plane, Cloud Build 2,500 min/month | Requires a **billing account with a card**; free limits are monthly and do not roll over; Cloud Run's 180k vCPU-s ≈ 50 h at 1 vCPU (25 h at 2 vCPU) — fine for demo renders; 1 GB egress means media must be served elsewhere; $300/90-day trial credit can be spent on overage | Card required; identity/bank verification possible; hardest of the mainstream clouds to keep strictly $0 by accident |
| **Cloudflare Workers** | Free: 100,000 requests/day, 10 ms CPU per request | 10 ms CPU cannot run FFmpeg; Workers Containers require the paid Workers plan ($5/month) | Easy signup; free tier fine for a lightweight API/edge layer, not the renderer |

**Not revisited here but worth knowing:** Hugging Face Spaces (free CPU tier with generous RAM) is occasionally used to host small FFmpeg demos; it was not verified in this pass and its ephemeral storage/sleep behavior is unsuitable for anything persistent.

---

## 4. Burst compute for FFmpeg (scale-to-zero)

| Provider | What's free | Caveats | Signup reality |
| --- | --- | --- | --- |
| **Modal** | **Starter: $0 + compute, incl. $30/month free credits** (recurring), 1 TiB/month egress, 100 containers, 10 GPU concurrency, 5 cron jobs, 1-day logs. Rates: CPU $0.0000131/core/s (~$0.047/core-h); RAM $0.00000222/GiB/s (~$0.008/GiB-h); volumes $0.09/GiB/mo | Credits are monthly, not rolled over; **not a general host** — best for jobs/functions triggered by HTTP/cron, not a long-running API; 1-day log retention on Starter; confirm Starter payment-method policy at signup | Self-serve; the friendliest "real compute" free tier for FFmpeg |
| **GCP Cloud Run** | 180k vCPU-seconds + 360k GiB-seconds + 2M requests/month, scale-to-zero | 1 GB NA egress only; requires billing account; long jobs must fit the request model | Card required (billing account) |

**Workload math (inference):** a 2 vCPU / 4 GiB FFmpeg job on Modal costs ≈ 2×$0.047 + 4×$0.008 ≈ **$0.126/hour**, so $30/month of credits ≈ **~230 hours of 2-core renders** — far more than a demo needs. On GCP, 180k vCPU-s is ≈ 25 hours at 2 vCPU, ≈ 50 hours at 1 vCPU.

---

## 5. Managed Postgres

| Provider | What's free | Caveats | Signup reality |
| --- | --- | --- | --- |
| **Neon** | **Free plan is permanent (not a trial):** 100 CU-hours/project/month (up to 2 CU / 8 GB RAM), 1 GB storage/project + 20 GB/account, 5 GB egress/project, 10 branches/project, Auth (60k MAU), 5 GB object storage/project, Functions (10 active capacity-h + 1M invocations/project), 6-h restore window | Scale-to-zero **after 5 min** (always on Free; can't disable) — first query after idle pays a cold-start; exceeding CU-hours or 5 GB egress **suspends compute until the next cycle** (data is kept); 1-day metrics retention | **Easiest: no credit card**; also a ["Claimable Neon"](https://neon.com/claimable-neon) instant project with **no signup at all** |
| **Aiven** | Free PostgreSQL/MySQL/OpenSearch/Kafka/Valkey: 1 CPU, 1 GB RAM, 1 GB storage, $0/month, **no time limit** | Can't select cloud/region; no connection pooling on free; single backup only; **Aiven reserves the right to shut down services "unused for an extended period"**; 1 GB storage is tiny | Email signup; plan is "demo/docs" tier |
| **Supabase** | Free: ~500 MB Postgres, ~1 GB file storage, ~5 GB egress, 50k MAU, 2 projects | **Projects pause after ~1 week of inactivity** (manual restore); 500 MB DB is small for media metadata; no PITR on free | Easy; no card |
| **Render Postgres (free)** | 256 MB RAM, 1 GB storage | **Expires 30 days after creation** — deleted after a 14-day grace unless upgraded; one per workspace; no backups; no connection pooling | Easy, but do not rely on it |
| **Koyeb Postgres** | **5 hours total** (0.25 vCPU / 1 GB / 1 GB) | Trial-length, not a free tier | Self-serve |

**Pick for this project:** Neon Free. Permanent, card-free, scale-to-zero, branch per environment, and it tolerates long idle periods. If an app frame wants auth/storage/realtime, Supabase Free is the runner-up — but treat the 7-day inactivity pause as a hard requirement, not a footnote.

---

## 6. Redis and async job transport

### 6a. The survey (Redis streams / RabbitMQ / managed queues)

| Option | Free allowance | Model | Fit for our pipeline |
| --- | --- | --- | --- |
| **Upstash QStash** | 1,000 messages/day, 50 GB/month bandwidth, 1 MB max message, 10 queues, queue parallelism 2 (max parallelism 10), 15-min HTTP response window, 3-day DLQ | **HTTP push** with retries, schedules, deduplication; language-agnostic — targets any webhook | **Best $0 default.** Free plan needs only an Upstash account (no card); works perfectly with a Modal/Render worker that receives a webhook. Parallelism caps mean batch renders queue politely |
| **Upstash Redis** | 250 MB data, 500k commands/month, 10 GB/month bandwidth, 1 database, 10k cmd/s, no SLA | Redis commands incl. **Streams** (XADD/XREADGROUP) over TLS; REST API available | Viable for `redis-rs` streams/queues at demo scale. 500k commands/month ≈ 16k/day — keep polling cheap (XREADGROUP with BLOCK); upgrade path is $0.20/100k commands. Instant free DB via `POST https://upstash.com/start-redis` (unclaimed DBs deleted after 3 days) |
| **CloudAMQP** | Free shared RabbitMQ "Little Lemur": 1M messages/month, 20 connections, 100 queues, **max idle queue time 28 days**; free shared LavinMQ "Loyal Lemming": 2M msgs/month, 40 connections, 200 queues, 20k queued-message cap | Real AMQP (RabbitMQ) — `lapin`/`amqprs` in Rust | Best when you want true AMQP semantics and dead-letter exchanges. Shared broker, no SLA; queues are purged after 28 days idle — fine for demo, not for durable state |
| **AWS SQS** | 1M requests/month (free tier) | Pull-based standard/FIFO queues | Solid and dull. Requires an AWS account + card; no push; payload 256 KB (claim-check pattern needed) |
| **GCP Pub/Sub** | 10 GiB of messages/month | Push or pull subscriptions | Works with Cloud Run push; requires billing account; good if the stack is already GCP |
| **Fly.io + Upstash extension** | None free | Redis provisioned via Fly, billed on Fly invoice | Only relevant if committing to Fly |
| **Trigger.dev** | Free: $5/month recurring credits, 20 concurrent runs | Durable background jobs/orchestration (TS SDK) | Good for TS monoliths; Rust worker would poll its API — awkward |
| **Inngest** | Free plan, no credit card | Durable event-driven functions | Same TS-centric caveat; concept ("events trigger steps") maps well if the orchestrator lives in TS |

### 6b. Recommendation for our stack

1. **Default:** QStash publishes `render.requested` → worker webhook (Modal/Render). $0, retries built in, no infra.
2. **If we need streams/backpressure:** Upstash Redis Streams (`XADD`/`XREADGROUP`), capped by command quotas; keep payloads in R2 and pass keys.
3. **If we need real AMQP:** CloudAMQP free Little Lemur, accepting the 28-day idle purge and shared-broker limits (demo only).
4. **At scale-up:** SQS or Pub/Sub as the durable backbone; QStash stays for scheduled/cron triggers.

---

## 7. Object storage and CDN

| Provider | What's free | Caveats | Signup reality |
| --- | --- | --- | --- |
| **Cloudflare R2** | 10 GB-month storage, 1M Class A ops, 10M Class B ops per month; **egress $0 forever** (incl. public buckets via `r2.dev`/Workers/S3 API) | Free tier applies to Standard class only; billing rounds up to next unit; storage billed on *daily peak average*; R2 activation **may require adding a payment method** to enable the subscription (free allowance itself is $0) — verify at signup | Self-serve Cloudflare account; card may be requested to enable R2 |
| **Backblaze B2** | **First 10 GB storage always free**; free egress up to **3× average monthly storage**; **unlimited free egress through CDN partners** (Cloudflare, Fastly, bunny.net, CacheFly, …); Class A/B/C API calls free | $0.00695/GB-month beyond 10 GB; egress beyond 3× is $0.01/GB; **Canada-country bucket nuance** — double-check free-egress partner routing rules for your bucket region | Easy, no card for the free allowance; card needed to exceed |
| **GCP Cloud Storage** | 5 GB-months regional storage (us-east1/us-west1/us-central1 combined), 5,000 Class A + 50,000 Class B ops, **100 GB/month egress from North America** (excl. China/Australia) | Requires billing account; small operation counts break under frequent listing/multipart; egress measured from NA only | Card required |
| **Storj** | 30-day free trial only (25 GB via Object Mount) | After trial: $7/TB storage + $7/TB egress, **$5/month minimum** — not a free tier | Trial no card, but useless as a long-term $0 store |
| **Oracle Object Storage** | 20 GB combined (10 Standard + 10 IA + 10 Archive while trial credits exist; 20 GB combined in Always-Free-only state), 50k API requests/month; paired with the 10 TB/month egress | S3 API is partially compatible; lives inside the famously painful Oracle signup; request counts are low for video chunk uploads | Requires Oracle account (card verification) |
| **Railway Object Storage** | $0.015/GB-month, **free egress** | Priced, not free (no free allowance published) | Easy but tied to Railway |

**Pick:** Cloudflare R2 as the primary (zero-egress, S3-compatible, tidy free ops limits); Backblaze B2 as the card-free fallback (10 GB free + 3× egress, and unlimited free egress when proxied through Cloudflare). Both keep us out of the egress row of every other table in this document.

---

## 8. Signup reality, ranked (what will actually block us)

1. **No card, minutes to start:** Neon (incl. claimable, no signup), Upstash (Redis/QStash; instant Redis via POST), CloudAMQP, Aiven, Supabase, Render free, Railway free trial, Backblaze B2 (within 10 GB), Modal (self-serve Starter).
2. **Card needed at signup:** Fly.io (after trial), Koyeb, GCP (billing account), AWS (SQS), Northflank (assumed; confirm).
3. **Hardest, plan a session for it:** Oracle Cloud (identity + card verification, region capacity lottery for A1 shapes, idle-reclamation policy), GCP if you want to avoid accidental billing.

---

## 9. Coherent full-stack shortlists at ~$0 idle

### Shortlist A — "Card-free assembly, sleeps between demos" (recommended default)

| Layer | Choice | Free allowance | Why |
| --- | --- | --- | --- |
| API (Rust container) | **Render free web service** | 0.1 CPU/512 MB, 750 h/month, Docker | Only free PaaS with real containers + custom domains; sleeps after 15 min — fine for demo |
| FFmpeg workers | **Modal Starter** | $30/month credits + 1 TiB egress | Scale-to-zero, ~230 h/month of 2-core renders, no idle cost |
| Postgres | **Neon Free** | 100 CU-h/project, 1 GB, permanent | Scale-to-zero, no card, no expiry |
| Queue | **Upstash QStash** (+ Redis if needed) | 1,000 msgs/day; 250 MB/500k cmds | HTTP push + retries; no card |
| Storage/CDN | **Cloudflare R2** (B2 as fallback) | 10 GB + $0 egress | Video egress is free forever |

**Idle cost: $0.** Sleeping: Render (15 min) and Neon (5 min) both park; QStash/R2 bill per use; Modal charges only while running.
**Hard rules learned:** do not put the demo Postgres on Render free (30-day expiry); do not have Render fetch large media (service-initiated traffic can be suspension-worthy); serve everything through R2.

### Shortlist B — "One always-on box" (no sleeps, no cold starts)

| Layer | Choice | Free allowance |
| --- | --- | --- |
| Everything | **Oracle Always Free ARM A1** (2 OCPU/12 GB, up to 2 instances), Docker: Rust API + FFmpeg + Postgres + Redis, 200 GB block volumes, 10 TB/month egress | Permanent (with idle-reclamation caveat) |

**Idle cost: $0.** Best if we hate cold starts and want a "real server". Risks: idle reclamation (the demo *is* idle most of the time — run a trivial keep-busy job), A1 capacity may be unavailable, and we manually operate Postgres/Redis/RabbitMQ. Backups and reproducibility are our problem.

### Shortlist C — "Scale-to-zero managed, slightly beyond demo"

| Layer | Choice | Free allowance |
| --- | --- | --- |
| API + workers | **GCP Cloud Run** (scale-to-zero containers) | 180k vCPU-s + 360k GiB-s + 2M req/month |
| Postgres | **Neon Free** | as above |
| Queue | **GCP Pub/Sub** (10 GiB/month) or QStash | as above |
| Storage/CDN | **Cloudflare R2** (or GCS 5 GB + 100 GB NA egress) | as above |

**Idle cost: $0** while scaled to zero, but this shortlist requires a GCP billing account and discipline: Cloud Run's 1 GB/month NA egress means serving video from Cloud Run burns money; media must come from R2. This is the natural graduation path when Shortlist A's credits or hours run out.

---

## 10. Bottom line for the decision

- **Build the demo on Shortlist A.** It needs no credit card anywhere in the critical path (verify R2 activation), costs exactly $0 while idle, and each layer has an independent upgrade path: Render → Fly ($2.19/mo) or Cloud Run when the API must stay warm; Neon → its paid per-CU-hour billing; QStash → Upstash Redis/SQS; R2 → stays R2, that's the point.
- **The egress decision is the architecture decision.** R2 (or B2 behind Cloudflare) makes video serving free; everything else in the stack is rounding error by comparison. Compute platforms with metered egress (Render, Northflank, Railway) must never host user-facing media.
- **Avoid the three classic traps:** (1) Render Free Postgres 30-day expiry; (2) sleeping APIs for anything that needs 24/7 webhooks — QStash retries cover short sleeps, but a 15-minute Render sleep on the *callback target* needs a wake-aware worker (Modal webhook endpoints or Cloud Run are better callback targets); (3) Oracle idle reclamation for a "set it and forget it" Always Free VM.
- **If we want one provider to blame**, Northflank Sandbox (2 always-on services + 1 DB + cron jobs) or Oracle Always Free are the only single-vendor $0 stories that don't sleep — budget a painful signup session for Oracle, and cap Northflank egress expectations at "demo only" ($0.06/GB).
- **Everything here is a demo tier.** Free tiers have no SLAs, no backups worth the name, and their terms change quarterly (Koyeb→Mistral, Fly's trial-only policy, Render's expiries are all recent examples). Treat this document as a 2026-10 snapshot and re-verify at the next binding decision (ticket #54).

---

## Sources

All fetched 2026-10-18.

- Render — Deploy for Free docs: https://render.com/docs/free
- Render — Pricing (Hobby bandwidth, compute/postgres/key-value plans): https://render.com/pricing
- Fly.io — Resource Pricing (no free tier, trial, $2.19 machine, egress): https://fly.io/docs/about/pricing/
- Railway — Pricing (Free $1/mo, trial $5 no card, egress $0.05/GB, object storage): https://railway.com/pricing
- Koyeb — Pricing (Pro $29, Postgres free 5h): https://www.koyeb.com/pricing
- Koyeb — joining Mistral AI announcement: https://www.koyeb.com/blog/koyeb-is-joining-mistral-ai-to-build-the-future-of-ai-infrastructure
- Northflank — Pricing (Sandbox tier, egress $0.06/GB): https://northflank.com/pricing
- Oracle — Always Free Resources docs (2 OCPU/12 GB A1, 200 GB block, 10 TB egress, idle reclamation, out-of-capacity): https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm
- Google Cloud — Free program docs (Free Tier limits, trial, eligibility): https://cloud.google.com/free/docs/free-cloud-features
- Cloudflare — R2 pricing (10 GB free, $0 egress, ops classes): https://developers.cloudflare.com/r2/pricing/
- Cloudflare — Workers pricing: https://developers.cloudflare.com/workers/platform/pricing/
- Cloudflare — Queues pricing: https://developers.cloudflare.com/queues/platform/pricing/
- Modal — Pricing (Starter $30 credits, 1 TiB egress, CPU/RAM rates): https://modal.com/pricing
- Neon — Pricing (permanent Free plan, 100 CU-h, scale-to-zero, claimable): https://neon.com/pricing
- Neon — Claimable Neon: https://neon.com/claimable-neon
- Supabase — Pricing: https://supabase.com/pricing
- Upstash — Redis pricing (250 MB / 500k commands / 10 GB, instant Redis): https://upstash.com/pricing/redis
- Upstash — QStash pricing (1k msgs/day, 50 GB, parallelism): https://upstash.com/pricing/qstash
- CloudAMQP — Plans (free Little Lemur RabbitMQ / Loyal Lemming LavinMQ): https://www.cloudamqp.com/plans.html
- AWS — SQS pricing (1M requests free tier): https://aws.amazon.com/sqs/pricing/
- Google Cloud — Pub/Sub pricing (10 GiB free tier): https://cloud.google.com/pubsub/pricing
- Trigger.dev — Pricing: https://trigger.dev/pricing
- Inngest — Pricing: https://www.inngest.com/pricing
- Backblaze — B2 pricing (10 GB free, 3× egress, $0.01/GB, partner CDNs): https://www.backblaze.com/cloud-storage/pricing
- Storj — Pricing (trial only, $7/TB + $7/TB, $5 min): https://www.storj.io/pricing
- Aiven — Pricing (free Postgres/MySQL/OpenSearch/Kafka/Valkey, 1 GB/1 CPU/1 GB VM): https://aiven.io/pricing
- Google Cloud — Cloud Run pricing: https://cloud.google.com/run/pricing
- Google Cloud — Cloud Storage pricing: https://cloud.google.com/storage/pricing
