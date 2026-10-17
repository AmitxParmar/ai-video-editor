# Research: Managed auth options — free tier and Rust integration (2026)

**Ticket:** [#52](https://github.com/AmitxParmar/ai-video-editor/issues/52)
**Date:** 2026-10-18
**Question:** Which managed auth providers have a meaningful free tier and Rust-friendly integration for a TypeScript/React SPA + Rust backend API (likely Axum)? Limits, ergonomics, lock-in, year-one cost (target $0).

**Context from this repo:** the design docs list "Build vs buy for authentication (managed IdP vs self-hosted)" as an open decision point (System Design Diagrams, "Decision Points"), with PostgreSQL as the system of record, authentication in the synchronous request path, and single-user project ownership in scope. This ticket resolves that decision point for the portfolio/`$0` constraint.

**Method:** vendor pricing pages and official docs fetched 2026-10-18; Rust crate facts from crates.io/docs.rs. Pricing changes often — re-verify before committing.

---

## TL;DR

- Every serious modern provider (Clerk, Auth0, Cognito, Supabase, WorkOS, Zitadel, Logto, Better Auth, Keycloak) issues **asymmetric-signed JWTs verified against a JWKS endpoint**. That means *one* piece of Axum middleware (bearer-token validation via `jsonwebtoken` + JWKS) works for all of them, and switching providers later is mainly a user-store migration, not a code rewrite.
- No vendor ships an official, fully supported **Rust SPA/backend SDK**. WorkOS is the exception for the *management/API* side via an officially published Rust crate; Clerk has an official-community Rust SDK; the rest are community crates or plain JWT validation.
- At `$0` for a single-user portfolio project, the shortlist is essentially **Clerk** (best React DX, custom domain on free) vs **Supabase Auth** (lowest lock-in, 50K MAU, OSS escape hatch) vs **Better Auth** (MIT, self-hosted, zero vendor). WorkOS AuthKit's 1M-MAU free tier is remarkable but enterprise-shaped.
- Free-tier gotchas that matter here: Clerk free has **no MFA/passkeys** and a **fixed 7-day session**; Supabase free **pauses projects after 7 days of inactivity**; Auth0 free has **no RBAC** and 1-day logs; Cognito's new-pool free tier is **10K MAU** (the old 50K tier only applies to pools created before 2024-11-22); Ory has **no free production environment** at all (`$770/yr` minimum).
- **No `$0` option is disqualified by the Rust side.** The decision should be made on SPA developer experience, free-tier headroom, and lock-in — not on Rust compatibility.

---

## Comparison table

| Provider | Free tier (2026) | Rust / Axum integration | Lock-in | Year-1 cost (single user) |
|---|---|---|---|---|
| **Clerk** | 50,000 MRU/app, unlimited apps, custom domain included, up to 3 social connections; **no MFA/passkeys**; fixed 7-day sessions; "Secured by Clerk" branding; 1-day logs; no credit card [1] | RS256 session JWTs; JWKS at Frontend API URL + `/.well-known/jwks.json` (or `api.clerk.com/v1/jwks`); validate `exp`/`nbf` **and** `azp` (authorized parties) to avoid CSRF [2]. No official Rust SDK; community `clerk-rs` v0.5.0 (154K downloads) for the management API [8] | Moderate: full user-data export; password hashes/user store are not portable | **$0** |
| **Auth0** | 25,000 MAU; 1 custom domain (credit card verification required); unlimited social connections; passkeys included; 5 orgs; **Role Management/RBAC not available on Free**; 1-day log retention [3] | Standard OIDC: RS256 access tokens validated via tenant `/.well-known/jwks.json` with any JWT library (`jsonwebtoken`). No official Rust SDK; old community `auth0` crate v0.3.0 [8] | Moderate–high: tenant config, Actions, Organizations modeling; no vendor lock at token layer | **$0** |
| **AWS Cognito** | Lite/Essentials: **10,000 MAU/month** per account (new pools; 50K free tier only for pools created before 2024-11-22); 50 MAU for SAML/OIDC federation; **no free tier on Plus**; identity pools free; M2M token requests have no free tier [4] | JWKS at `https://cognito-idp.{region}.amazonaws.com/{userPoolId}/.well-known/jwks.json`; RS256; check `iss`, `exp`, `token_use`, `aud` (ID token) / `client_id` (access token); cache keys by `kid`; revocation not detectable offline [5]. No official Rust SDK | Moderate–high: AWS ecosystem gravity; tokens are OIDC-ish but not pure OIDC | **$0** |
| **Supabase Auth** | 50,000 MAU; unlimited API requests; social OAuth; OAuth 2.1 server; custom SMTP; basic MFA included; **no branding removal**; custom domain is a paid add-on ($10/domain/mo); 1-hour auth audit logs; free projects **paused after 7 days inactivity** (restorable within 1 year, data retained) [6][16] | Asymmetric signing keys → JWKS at `https://<project>.supabase.co/auth/v1/.well-known/jwks.json` (RS256/ES256); docs recommend a JWT verification library and warn against HS256 shared-secret validation [7]. No official Rust SDK; verification is plain JWKS + `jsonwebtoken` | **Low**: MIT/Apache OSS, Docker self-hosting, data in your Postgres | **$0** |
| **WorkOS AuthKit** | **First 1,000,000 MAU free** (then $2,500 per additional 1M MAU); email+password, social, passkeys, MFA, magic auth, enterprise SSO in the free AuthKit; **custom domain $99/mo**; SSO connections $125/ea; staging free [9] | Access token JWT; JWKS at `api.workos.com/sso/jwks/<clientId>`; claims `sub`, `sid`, `iss`, `org_id`, `role`, `permissions`, `exp`; Next/Remix SDKs automate it, other backends (e.g. Rust) validate + refresh manually [10]. **Official Rust SDK** (`workos` crate v3.4.1, 850K downloads, published by `workos-sdk`) for the API side [11] | Moderate: WorkOS user store, but standards-based tokens; enterprise-oriented shape | **$0** (custom domain extra) |
| **Zitadel** | Free cloud: unlimited users stored, **100 Daily Active Users** (not MAU), 3 IdPs, unlimited orgs, all security features, EU/US/CH/AU residency; custom domain only on Pro ($100/mo) [12] | OIDC-certified provider → standard JWKS validation with `jsonwebtoken`; no official Rust SDK | **Low**: AGPL OSS, self-hostable, standard OIDC | **$0** |
| **Logto** | 50,000 MAU; 50K access tokens/mo; 3 applications; 1 M2M app; **1 custom domain included**; 3 social connectors; 3-day audit logs; MFA/RBAC/enterprise SSO are paid add-ons; Pro from $24/mo [13] | Standard OIDC JWKS validation; unofficial Rust SDKs only (`logto-rs` v0.1.0, `logto-sdk` v0.1.3) [8] | **Low**: MPL-2.0 OSS, self-hosted plans exist | **$0** |
| **Better Auth** | Framework is **MIT, free**. You host the auth server (TS runtime) and DB yourself. Managed infra (dashboard only) Starter $0: 1 seat, 10K audit logs/mo, 1K security detections/mo [14] | JWT plugin issues **EdDSA (Ed25519)** tokens (configurable ES256/RS256/…), JWKS at `/api/auth/jwks`, default 15-min expiry, key rotation with grace period; verify offline in Rust with `jsonwebtoken` 11 `rust_crypto`/`aws_lc_rs` features [15][17]. No vendor API to call | **Lowest**: MIT, self-hosted, users live in your DB | **$0** + hosting (an always-on TS service) |
| **Keycloak** | Free OSS (Apache-2.0), latest 26.8.1 (2026-10-13). No first-party managed free tier — self-hosted only [18] | Standard OIDC JWKS validation; no official Rust SDK | **Lowest**: Apache-2.0, fully self-hosted | **$0** but ops-heavy (JVM + DB + upgrades) |
| **Ory** | Developer plan free = 2 dev environments, **0 production environments**. Production from **$770/year** (~$64/mo), Growth $9,350/year [19] | Standard OIDC; self-hosted OSS components (Hydra/Kratos) are free | Low (OSS) but managed path is paid | **Not $0 for production** |

**Rust tooling used across the board** (candidate default stack):

- [`jsonwebtoken`](https://crates.io/crates/jsonwebtoken) **v11.1.0** (updated 2026-09-16; 214M downloads) — RS256/ES256/EdDSA via `rust_crypto` or `aws_lc_rs` features; `jwk::JwkSet` parsing [17].
- [`jwks`](https://crates.io/crates/jwks) v0.5.3 (2026-02) — fetching/parsing/caching JWK sets; [`axum-jwks`](https://crates.io/crates/axum-jwks) v0.12.0 (2025-06) — Axum extractor for JWKS-verified requests [8].
- An `openidconnect`-style crate is **not required** for the SPA + API pattern below; the SPA performs the OIDC flow and the Axum API only validates bearer tokens.

---

## Common integration pattern (works with every shortlisted provider)

1. **SPA (React/TS):** Authorization Code + PKCE against the provider (each vendor's React SDK or a generic OIDC client). Store the access token in memory; send it as `Authorization: Bearer <jwt>`.
2. **Axum API:** middleware fetches (and caches) the provider's JWKS, matches `kid`, validates signature + `iss`/`aud`/`exp` with `jsonwebtoken`, then maps `sub` → user. Provider-specific extras: Clerk requires the `azp` check [2]; Cognito uses `client_id` on access tokens instead of `aud` and a `token_use` check [5]; Supabase recommends an asymmetric key + library validation rather than the HS256 shared secret [7].
3. **Authorization:** per-project ownership checks in Postgres. None of the `$0` providers should be relied on for app-level RBAC: Auth0 free has no role management [3]; Clerk free has no org role sets beyond basic; Supabase free has no SAML SSO [6]. Keep authz in the database keyed by `sub`.

Implication: the Rust integration cost is roughly identical everywhere, so **lock-in lives in the provider's user store and management APIs, not in token validation**.

---

## Provider notes (decision-relevant details)

- **Clerk** — strongest React DX of the group; free tier uniquely includes a custom domain and 50K "monthly retained users" (users who return after 24h; one-time signups are not billed) [1]. Free-tier gaps: no MFA/passkeys, fixed 7-day session lifetime, Clerk branding on prebuilt UIs, 1-day logs. Rust validation is documented manual JWKS/RS256 with an `azp` check [2]; `clerk-rs` covers server-side API calls [8]. For a single-user portfolio this is the most "drop-in" choice, at the cost of no MFA on free.
- **Supabase Auth** — best lock-in profile: OSS, self-hostable, users in your own Postgres, 50K MAU free, and asymmetric JWTs verifiable with plain JWKS [6][7]. Caveats: no custom domain or branding removal on free; **free projects pause after a week of low activity** (restorable; data retained) [16] — a real risk for a demo site that sits idle, though a few daily requests prevent it.
- **Better Auth** — MIT and self-hosted, so effectively unlimited and no vendor bill; the trade-off is running a TS auth service (Node/Bun/Workers) alongside the Rust API, plus your own DB and email delivery [14]. The JWT plugin was designed exactly for non-session backends ("services that can't use the session") and gives you JWKS + Ed25519 for offline Rust verification [15].
- **WorkOS AuthKit** — 1M MAU free is far beyond the others, includes passkeys/MFA/SSO, and is the only vendor with an official Rust SDK (for the API, not token validation) [9][11]. It is shaped for B2B/enterprise (organizations, directory sync); custom domain costs $99/mo, so at `$0` users sign in on a WorkOS domain. Reasonable fallback if the project later needs SSO.
- **Auth0** — 25K MAU free, passkeys included, custom domain on free (credit card verification required), but **no RBAC on free**, 1-day logs, and the migration path to paid ($35/mo Essentials, $240/mo Professional, B2C) is steep [3]. Fine for `$0`, least attractive for a brand-new project.
- **Cognito** — free tier shrank for new pools to 10K MAU; SAML/OIDC federation is only 50 MAU free; Plus has none [4]. Validation is doable with `jsonwebtoken` (JWKS, RS256) but you must handle Cognito's access-token quirks (`client_id`, `token_use`) and its revocation caveats [5]. Best only if deliberately all-in on AWS.
- **Zitadel** — free cloud covers unlimited stored users but only **100 daily active users**; for a single-user portfolio that's plenty, and it includes all security features [12]. If the demo ever gets real traffic, Pro is $100/mo. Self-hostable (AGPL).
- **Logto** — 50K MAU + 1 custom domain free and an OSS (MPL-2.0) self-host option; MFA is a paid add-on ($48), which matters more than MAU for this project [13].
- **Keycloak** — the "never pay" option, but not managed: you run the JVM service and its database and upgrades [18]. Contradicts the spirit of "managed auth"; list it as the self-host fallback, not a shortlist pick.
- **Ory** — no `$0` production option ($770/yr minimum for one production environment) [19]. Excluded by the constraint.

---

## Bottom line for the decision

For a **single-user portfolio project targeting `$0`** with a TS/React SPA and a Rust (Axum) API:

1. **Recommended: Clerk (free Hobby plan).** Best SPA DX, 50K retained-user headroom, and a custom domain included at `$0` [1]. The Axum side is routine RS256/JWKS validation with the documented `azp` check [2]. Accept the free-tier trade-offs: no MFA/passkeys, fixed 7-day sessions, Clerk branding. This is the fastest path to a polished demo and the one with the least auth code to write.
2. **Best low-lock-in alternative: Supabase Auth** — pick this if the project adopts Supabase for Postgres (the design's system of record), or if an OSS self-hosting escape hatch matters more than DX polish. 50K MAU free, asymmetric JWKS, plain `jsonwebtoken` validation [6][7]. Mitigate the 7-day project-pause with a trivial keep-alive (or accept manual resume) [16]. Note the missing custom domain on free.
3. **Zero-vendor option: Better Auth** — choose only if running an extra TS auth service is acceptable; then auth is MIT, free forever, users in your DB, and Axum verifies Ed25519 JWTs via the JWT plugin's JWKS [14][15].
4. **Keep in reserve:** WorkOS AuthKit (1M MAU free + official Rust SDK, if SSO/enterprise features ever creep in) [9][11]; Logto (50K MAU + custom domain + OSS, but MFA is paid) [13]. Avoid for this project: Ory (no `$0` production), Keycloak (not managed), Cognito (10K MAU, AWS gravity), Auth0 (no RBAC on free), Zitadel (100 DAU headroom).

**Decision rule that falls out of the research:** all candidates validate identically in Rust, so choose on SPA experience and lock-in comfort; if either Clerk's missing free-tier MFA or Supabase's project pausing is a dealbreaker, the other is the correct answer.

---

## Sources

1. Clerk pricing (plans, MRU definition, Hobby limits): https://clerk.com/pricing
2. Clerk — Manual JWT verification (JWKS endpoints, RS256, `azp`): https://clerk.com/docs/guides/sessions/manual-jwt-verification
3. Auth0 pricing (Free plan limits, RBAC not on Free, paid tiers): https://auth0.com/pricing
4. AWS Cognito pricing (free tier 10K MAU new pools, 50 MAU federation, no Plus/M2M free tier): https://aws.amazon.com/cognito/pricing/
5. AWS docs — Verifying JSON web tokens (JWKS URI, RS256, claims to check): https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html
6. Supabase pricing (Free: 50K MAU, no custom domain/branding removal, 1h audit logs): https://supabase.com/pricing
7. Supabase docs — JWTs (asymmetric signing keys, JWKS endpoint, warning against HS256): https://supabase.com/docs/guides/auth/jwts
8. crates.io / docs.rs crate facts — `jwks` 0.5.3 (https://crates.io/crates/jwks), `axum-jwks` 0.12.0 (https://crates.io/crates/axum-jwks), `clerk-rs` 0.5.0 (https://crates.io/crates/clerk-rs), `auth0` 0.3.0 (https://crates.io/crates/auth0), `logto-rs`/`logto-sdk` (https://crates.io/crates/logto-rs)
9. WorkOS pricing (AuthKit 1M MAU free, custom domain $99/mo, SSO/Directory Sync pricing): https://workos.com/pricing
10. WorkOS docs — Sessions (access token JWT, JWKS location, claims): https://workos.com/docs/user-management/sessions
11. WorkOS official Rust SDK — `workos` v3.4.1 (crate owner `workos-sdk`): https://crates.io/crates/workos
12. Zitadel pricing (free cloud: unlimited users, 100 DAU, 3 IdPs; Pro $100/mo): https://zitadel.com/pricing
13. Logto pricing (Free: 50K MAU, 1 custom domain, 3 apps; add-ons; Pro $24/mo): https://logto.io/pricing
14. Better Auth pricing (framework MIT; infra Starter free; Pro $20/mo): https://www.better-auth.com/pricing
15. Better Auth docs — JWT plugin (JWKS endpoint, EdDSA default, rotation, issuer/audience/expiry): https://www.better-auth.com/docs/plugins/jwt
16. Supabase docs — Project pausing (7-day inactivity pause, 1-year restore, data retained): https://supabase.com/docs/guides/platform/project-pausing
17. `jsonwebtoken` v11.1.0 (2026-09-16; features `rust_crypto`/`aws_lc_rs`; 214M downloads): https://crates.io/crates/jsonwebtoken · https://docs.rs/crate/jsonwebtoken/latest/features
18. Keycloak (Apache-2.0; latest release 26.8.1, 2026-10-13): https://github.com/keycloak/keycloak/releases
19. Ory pricing (Developer = no production environments; Production $770/yr; Growth $9,350/yr): https://www.ory.sh/pricing/

Repository context: `docs/AI Video Editor_ Design Document Set.md` and `docs/AI Video Editor + Cloud Media Pipeline_ System Design Diagrams.md` (open decision: "Build vs buy for authentication").
