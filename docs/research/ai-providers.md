# AI Provider Landscape — Cheapest Viable Stack (Oct 2026)

Research for [#51](https://github.com/AmitxParmar/ai-video-editor/issues/51). Goal: pick providers for the three AI jobs in this app while staying on free tiers or a near-zero monthly bill.

**What the app actually needs:**

1. **Text reasoning + tool calling** — an agent that edits video timelines through tool calls (clip cuts, arrangement, project state queries). Needs reliable structured function-calling and enough reasoning to sequence edits.
2. **Transcription** — batch transcription of uploaded footage, ideally with word-level timestamps so captions/cuts can be placed precisely; diarization nice to have.
3. **Vision** — occasional frame understanding for scene detection. Not a hot loop; should be token-metered and cheap.

**Assumed portfolio volume** (used for cost math below): ~10 h of footage transcribed/month, ~1M input + 0.3M output tokens/month for the agent, ~500 frame analyses/month.

---

## 1. Text reasoning + tool calling

| Provider | Model | Free tier | Input $/M tok | Output $/M tok | Notes |
| --- | --- | --- | --- | --- | --- |
| Google | **Gemini 3.8 Flash** | ✅ **Yes** | $0.75 (promo → $1.50 after 2026-12-31) | $3.75 (promo → $7.50) | Tool calling, long context; promo pricing through 2026-12-31 |
| Google | **Gemini 3.5 Flash-Lite** | ✅ **Yes** | $0.30 | $2.50 | Cheapest paid Google text model |
| Google | Gemini 4 Argon | ❌ No | $2.00 (promo → $4.00 after 2026-11-30) | $10.00 (→ $20.00) | Flagship; not on the free tier |
| Groq | **gpt-oss-120b** | ✅ Free plan | $0.15 | $0.60 | Free plan: 30 RPM / 1K RPD / 8K TPM / 200K TPD |
| Groq | gpt-oss-20b | ✅ Free plan | $0.075 | $0.30 | Cheapest tool-capable option anywhere |
| DeepSeek | deepseek-v4-flash | ❌ No | $0.15 off-peak / $0.30 peak | $0.60 off-peak / $1.20 peak | Cache-hit input $0.003/M; 1M context; tool calls + vision |
| OpenAI | gpt-6-luna | ❌ No API free tier | $0.10 | $0.50 | Chat product is free; API is not |
| OpenAI | gpt-5-nano | ❌ No API free tier | $0.05 | $0.40 | |
| Anthropic | **Haiku 5.5** | ❌ (small new-user credits) | $0.10 (≤100k prompt; $0.50 above) | $0.50 (≤100k; $2.50 above) | Long prompts get 5× more expensive |
| Anthropic | Sonnet 5 | ❌ | $2.00 | $10.00 | Overkill for timeline edits |
| Mistral | Small 4 | ❌ | $0.15 | $0.60 | |
| Mistral | Ministral 3 8B | ❌ | $0.15 | $0.15 | Cheap output, small model |
| OpenRouter | free model variants | ✅ Free | $0 | $0 | 20 RPM; 50 RPD with <$10 lifetime credits, 1,000 RPD at ≥$10 |

**Read on this category:** Gemini free tier is the only free tier that covers a *strong* tool-calling model (3.8 Flash). Groq's free plan also works and is genuinely free without a card, but 8K TPM is tight for agent loops with long context — fine for short turns, painful for big transcripts in-context. Everything else is paid-only, and the paid floor is astoundingly low: Haiku 5.5, GPT-6 Luna, and off-peak DeepSeek flash are all ~$0.10/$0.50 or cheaper per million tokens.

**Caveat that matters here:** the Gemini free tier is covered by the *Unpaid Services* terms — content is used to improve Google products and may be human-reviewed. Do not send unreleased or private footage through it. Also, Google's terms require *Paid Services* for API clients made available to users in the EEA, Switzerland, or the UK. Groq, by contrast, does not train on API data by default and offers a zero-retention toggle.

---

## 2. Transcription

| Provider | Model | Price | Diarization | Word timestamps | Free allowance |
| --- | --- | --- | --- | --- | --- |
| Groq | whisper-large-v3-turbo | **$0.04/hr** | ❌ | ✓ | Groq free plan applies; 10 s minimum billing |
| Groq | whisper-large-v3 | $0.111/hr | ❌ | ✓ | Same |
| Google | Gemini 3.5 Transcribe | ~$0.005/min blended (~$0.30/hr) | ✓ | ✓ | ✅ Free tier |
| Deepgram | Nova-3 | $0.0043/min mono (~$0.26/hr) | ✓ | ✓ | **$200 signup credit (~43k min), no credit card**; EU endpoint available |
| AssemblyAI | Universal-2 | $0.15/hr | +$0.02/hr add-on | ✓ | $50 free credits; EU region same price |
| ElevenLabs | Scribe v2 | $0.22/hr batch; $0.39/hr realtime | ✓ | ✓ | Free/PAYG allowance ≈ 4h 30m — *verify at source* |
| Mistral | Voxtral Mini Transcribe 2 | $0.003/min (~$0.18/hr) | — | ✓ | None |
| OpenAI | gpt-4o-mini-transcribe | $0.003/min | — | — | None; whisper-1 $0.006/min |
| xAI | STT | $0.10/hr REST; $0.20/hr streaming | — | ✓ | None |

**Read on this category:** whisper-large-v3-turbo on Groq at $0.04/hr is the price/perf champion for batch transcription — 10 hours of footage costs **$0.40**. Its weakness is no diarization. If diarization matters, Deepgram Nova-3 is the pragmatic pick: the $200 signup credit (no credit card) covers ~43,000 minutes ≈ 700+ hours — effectively free for this project's lifetime — and `mip_opt_out=true` gets zero data retention. Note Groq bills a 10-second minimum per clip, so don't shatter uploads into sub-10-second chunks.

---

## 3. Vision / frame understanding

| Provider | Model | Free tier | Price | Notes |
| --- | --- | --- | --- | --- |
| Google | **Gemini 3.8 Flash / 3.5 Flash-Lite** | ✅ **Yes** | Token-based; same rates as text | Strong frame Q&A; free tier data caveat applies |
| Groq | qwen/qwen3.8-27b | ✅ Free plan | $0.80 / $4.00 per M tok | 2,048 tokens/image, max 3 images/request; tool use with images |
| OpenAI | gpt-6-luna / gpt-5-nano / 4o-mini | ❌ | $0.10/$0.50 etc. | |
| DeepSeek | deepseek-v4-flash | ❌ | $0.15/$0.60 off-peak | Vision included in the flash model |
| Mistral | OCR 4.1 | ❌ | $4 / 1,000 pages | Document OCR, not scene-framing Q&A |
| fal / Replicate | any open model | ❌ | Per GPU-time (Replicate: T4 $0.81/hr, H100 $5.49/hr) | Not cost-effective for light, sporadic frame analysis |

**Read on this category:** frame Q&A is a token problem, not a GPU problem — serverless-GPU hosts make no sense at this volume. Gemini free tier (or paid Flash-Lite at $0.30/$2.50) or Groq's Qwen at $0.80/$4.00 with the free plan. At ~500 frames/month and ~1.3k tokens/frame, this is well under $1/month even on the paid paths.

---

## 4. Privacy & retention (who sees the footage?)

| Provider | Trains on API data? | Retention | Notes |
| --- | --- | --- | --- |
| Google Gemini — **free tier** | ⚠️ **Yes** — improves products, human review possible | Unpaid terms | Do not use for private footage |
| Google Gemini — **paid tier** | No | Limited abuse logs only | Required for API clients serving EEA/CH/UK users |
| Groq | No by default | Zero-retention toggle; 30 d abuse logs; US GCP | |
| OpenAI | No | 30 d abuse logs; audio transcription endpoints excluded from abuse retention | |
| Anthropic | No by default | Deleted ≤30 d; ZDR agreement available | |
| Deepgram | No | **Opt-in** — MIP retains by default; `mip_opt_out=true` = zero retention; 90 d metadata | EU endpoint available |
| AssemblyAI | No | EU region available at same price | |
| OpenRouter | Per upstream provider | Free variants have logging conditions — check before routing sensitive content | Provider routing/ZDR controls documented |

---

## 5. Cost at portfolio volume

Assumptions: 10 h transcription, 1M in + 0.3M out agent tokens, 500 frames/month.

| Capability | Free-tier pick | Monthly | Paid fallback pick | Monthly |
| --- | --- | --- | --- | --- |
| Agent (tools) | Gemini 3.8 Flash free | $0 | Haiku 5.5 / GPT-6 Luna | ~$0.25 |
| Transcription | Groq whisper turbo (free plan) | $0 | Groq whisper turbo paid | $0.40 |
| Vision | Gemini 3.5 Flash-Lite free | $0 | Gemini Flash-Lite paid / Groq Qwen | ~$0.50 |
| **Total** | | **$0** | | **≈ $1–3** |

The Deepgram $200 credit would make transcription effectively free for ~700 hours of footage, unbounded by free-tier rate limits.

---

## Bottom line for the decision

**Cheapest viable stack — $0/month:**
- **Agent + tool calls:** Gemini **3.8 Flash** on the free tier (`gemini-3.8-flash`), with **Groq gpt-oss-120b** on the free plan as a backstop when Gemini rate limits bite.
- **Transcription:** **Gemini 3.5 Transcribe** free tier for primary (diarization + word timestamps included); **Groq whisper-large-v3-turbo** free-plan as the high-throughput batch path.
- **Vision:** Gemini **3.5 Flash-Lite** free tier for sampled frames.
- **Provider interface matters more than the pick:** design three thin ports — `text/tools`, `transcribe`, `vision` — so the concrete provider is a config switch. Free tiers move; the app shouldn't.

**Why not "just Groq" as primary?** Free Gemini gives a stronger reasoning model for timeline edits at $0, plus transcription *with* diarization and word timestamps. Groq is the better *fallback*: no training on data by default, higher transcription throughput, and genuinely free without a credit card.

**Credible fallback — ≈ $1–3/month at portfolio volume:**
- **Agent:** Anthropic **Haiku 5.5** ($0.10/$0.50 ≤100k ctx) *or* OpenAI **GPT-6 Luna** ($0.10/$0.50) *or* **DeepSeek v4 flash** off-peak ($0.15/$0.60, 1M ctx, cache hits at $0.003/M).
- **Transcription:** **Groq whisper-large-v3-turbo** at $0.04/hr ($0.40 for 10 h) — or **Deepgram Nova-3** backed by the $200 signup credit if diarization/retention guarantees matter.
- **Vision:** **Gemini 3.5 Flash-Lite** paid ($0.30/$2.50) or **Groq Qwen 3.8 27B** ($0.80/$4.00).

**Switch trigger:** the moment footage can't touch the Gemini free tier (private/unreleased content, EEA/UK users) or free-tier rate limits start failing agent turns in production. The paid fallback removes both constraints for roughly the price of a coffee per year at this volume.

## Risks & watch items

1. **Gemini free-tier data usage + human review** — the single biggest reason to keep a non-Google fallback wired up from day one.
2. **Promo pricing ends soon** — Gemini 3.8 Flash promo through **2026-12-31**, Argon promo through **2026-11-30**. Budgets shift at the turn of the year.
3. **Gemini free-tier rate limits are model-specific and published per-account** — the static rate-limit table is no longer on the docs page; check the live AI Studio rate-limit view.
4. **Groq free plan: 8K TPM** — fine for agent turns, tight for long transcripts in-context; batch/chunk accordingly.
5. **Groq whisper: 10-second minimum billing, no diarization.**
6. **EEA/CH/UK**: Google ToS requires *Paid Services* for API clients made available to users there.
7. **ElevenLabs free allowance (~4h 30m)** copied from the pricing page but flagged *verify at source*.
8. **OpenRouter free variants**: confirm per-variant logging conditions before sending footage.

## Sources

- Gemini pricing: https://ai.google.dev/gemini-api/docs/pricing
- Gemini rate limits: https://ai.google.dev/gemini-api/docs/rate-limits
- Gemini terms of service: https://ai.google.dev/gemini-api/terms
- Anthropic pricing: https://platform.claude.com/docs/en/about-claude/pricing
- Anthropic privacy (consumer): https://privacy.claude.com/en/articles/7996866
- Anthropic privacy (commercial/API): https://privacy.claude.com/en/articles/7996868
- OpenAI pricing: https://platform.openai.com/docs/pricing
- OpenAI data usage: https://developers.openai.com/api/docs/guides/your-data
- Groq rate limits / free plan: https://console.groq.com/docs/rate-limits
- Groq models & pricing: https://console.groq.com/docs/models
- Groq data policy: https://console.groq.com/docs/your-data
- Groq vision docs: https://console.groq.com/docs/vision
- OpenRouter limits: https://openrouter.ai/docs/api-reference/limits
- OpenRouter privacy & logging: https://openrouter.ai/docs/features/privacy-and-logging
- Deepgram pricing: https://deepgram.com/pricing
- Deepgram data policy: https://developers.deepgram.com/trust-security/your-data
- AssemblyAI pricing: https://assemblyai.com/pricing
- ElevenLabs API pricing: https://elevenlabs.io/pricing/api
- Mistral pricing: https://docs.mistral.ai/inference/pricing
- DeepSeek pricing: https://api-docs.deepseek.com/quick_start/pricing
- xAI models & pricing: https://docs.x.ai/docs/models
- fal pricing: https://fal.ai/pricing
- Replicate pricing: https://replicate.com/pricing
