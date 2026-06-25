# Niilox platform — progress & what's left

**Last updated:** June 2026  
**Production API:** `https://api.driftin.live` · **Portal:** [admin.driftin.live](https://admin.driftin.live)

This document tracks the **full stack** — Go API, Rust media-svc, frontends, SDK, and ops — not just the TypeScript client.

---

## At a glance

| Layer | Status | Notes |
|-------|--------|-------|
| **Go API (`backend/drift`)** | **Production** | Multi-tenant, native auth, 59 migrations |
| **Postgres + PgBouncer** | **Production** | PG 18, replica, backup drills passed |
| **LiveKit (livestream)** | **Production** | Cloud + JHB region; Drift tenant |
| **Peer / P2P signaling** | **Production** | Rodent wire protocol; GeoGig + Rodent tenants |
| **Rust media-svc** | **Optional** | Quality cop on LiveKit; enable with `MEDIA_SVC_ENABLED` |
| **Redis WS fanout** | **Optional** | Set `REDIS_URL` before multi-replica scale |
| **Drift web (`frontend/drift-web`)** | **Production** | Livestream product on Vercel |
| **GeoGig (`geogig/`)** | **Production API** | Expo app; partial SDK adoption |
| **Admin portal (`drift-admin/` → mango)** | **Rebranded** | Niilox API; Vercel deploy may lag |
| **`@niilox/sdk`** | **v0.1 beta** | Monorepo only; not on npm |
| **Public docs mirror** | **Synced** | `locust-api-docs/` → GitHub `Dr-M06/locust-api` |

---

## Production infrastructure (done)

| Item | Status |
|------|--------|
| Hetzner API host (`api.driftin.live`) | Done |
| Native auth (no Supabase) | Done — [MIGRATION_STATUS.md](./MIGRATION_STATUS.md) |
| Google, magic link, phone OTP (Africa's Talking), Apple, password | Done |
| Bunny.net storage (avatars, DMs, ID docs, thumbnails, backups) | Done |
| Stripe + Flutterwave webhooks | Done |
| RevenueCat mobile webhook | Done |
| Daily encrypted Postgres backup → Bunny | Done |
| Failover / failback drills (May 2026) | Done |
| LiveKit webhooks + room reconcile worker | Done |
| Per-tenant `apps` row (LiveKit, Bunny, payment keys) | Done |
| Rate limiting (per IP + per `X-App-ID`) | Done |
| Admin MFA (TOTP) + WebAuthn routes | Done |
| Developer portal + API keys (`/platform/*`) | Done |

### Ops optional / verify

| Item | Status |
|------|--------|
| `REDIS_URL` for horizontal WS | Not required at current scale |
| `MEDIA_SVC_ENABLED=true` in prod | Optional — stream quality warnings |
| Quarterly restore drill | Operator task |
| Cancel legacy Supabase subscription | Cost savings only |

---

## Backend API — shipped by domain

All routes live under `/api/v1` unless noted. Source: `backend/drift/cmd/server/main.go`.

### Core platform

| Domain | Routes / behaviour | SDK module |
|--------|-------------------|------------|
| Health | `GET /health` | — |
| Auth | Guest, Google, magic, phone, password (+ forgot/reset), Apple, refresh, logout, admin MFA, WebAuthn | `auth` |
| Users | Profile, avatar, socials, discover, token balance | `users` |
| Platform (B2B) | Ping, tenants, keys, usage, business, token-packs, test-sign | `platform` |
| Notifications | Feed, unread, mark read | `notifications` |
| Push (native) | Expo token register, web push subscribe | `push` |
| Announcements | Admin → bell feed | `announcements` |

### Livestream (`drift` tenant) — **server / LiveKit only**

| Domain | Status | SDK |
|--------|--------|-----|
| Rooms (create, join, leave, list, end) | Done | `rooms` |
| Room chat + delete own message | Done | `rooms` |
| VIP token paywall + **capped seats** | Done | `rooms` + **`seats`** module |
| Fiat VIP / gift checkout | Done | `payments.fiatRoomAccess`, `payments.fiatGift` |
| Stage (guest co-hosts, mute) | Done | `stage` |
| Moderation (mute, kick) | Done | `moderation` |
| Gifts + token balance + gifters | Done | `gifts` + `users.tokenBalance` |
| Room thumbnails (host JPEG upload) | Done | `rooms.uploadThumbnail` |
| Scheduled sessions | Done | `scheduled` |
| LiveKit webhooks | Done | — (server only) |
| Rust quality cop (`media-svc`) | Done (opt-in deploy) | — |

### Social & wallet (all tenants)

| Domain | Status | SDK |
|--------|--------|-----|
| Follow graph | Done | `users` |
| DMs + media + typing + delete | Done | `dms` |
| Wallet, payouts, withdrawals, `/banks` | Done | `wallet` |
| Token packs + web checkout | Done | `payments` |

### Identity & trust

| Domain | Status | SDK |
|--------|--------|-----|
| ID verification queue (doc + selfie, admin approve) | Done | `verification` |
| Country catalog + expiry reminders | Done | `verification` |
| Nigeria bank identity (₦100 Flutterwave) | Done | `payments` + `profile` |
| GeoGig profile, mode, CV, trust, refund | Done | `profile` |
| Host vs worker verification gates on gigs | Done | — |

### GeoGig (`X-App-ID: geogig`)

| Domain | Status | SDK |
|--------|--------|-----|
| Gigs CRUD + lifecycle (accept, cancel, dispute, no-show) | Done | `gigs` |
| Geofence check-in/out + location trail | Done | `gigs` |
| Gig fiat checkout (Stripe / Flutterwave) | Done | `payments` |
| Reviews + ratings | Done | `gigs` |
| Categories, chef fields, currency | Done | `gigs` |
| P2P video glance (`POST /gigs/{id}/video-call`) | Done | `gigs` + `peer` |
| Worker safety | Done | `safety` — guide on request |
| Negotiation / claim flow | Done (app) | — |
| Push notifications (Expo) | Done | `push` (token register; server send is API-key only) |
| Gig image upload (multipart) | Done | `gigs.uploadImage` |

### Rodent (`X-App-ID: rodent`)

| Domain | Status | SDK |
|--------|--------|-----|
| Peer signaling (`/peer/ice`, `/peer/signal`) | Done | `peer` |
| Peer devices + doorcam (owner) | Done | `peer` (devices, decline; not cam firmware session) |
| Drop-off encrypted files (`/drop/*`) | Done | `drop` |
| Pairings + paid bookings | Done | `rodent` |
| Rodent Pro + token packs | Done | `payments` + `platform` |
| Paid bookings + Stripe checkout | Done | `payments.fiatBooking` + `rodent.createBooking` |
| Google Calendar sync | Done | `integrations` |
| Scheduled streams + reminders | Done | `scheduled` |
| WebAuthn passkeys | Done | `auth.webauthn*` |

### Other tenants

| Tenant | Status | SDK |
|--------|--------|-----|
| `rabbaly` | Routes wired | `rabbaly` |
| Custom tenants | Provision via admin portal | `platform` |

---

## Media architecture (implemented)

| Use case | Technology | Products |
|----------|------------|----------|
| **Public livestream** | LiveKit SFU + `/rooms/*` | Drift |
| **1:1 video / P2P chat** | WebRTC + `/peer/*` | GeoGig, Rodent |
| **DM history + typing** | REST + `/ws/me` | All |
| **Capped inventory** (VIP, events) | Server-enforced + WS lobby updates | Drift + B2B (`client.seats`) |
| **Stream quality on bad networks** | Rust `media-svc` on LiveKit | Drift (optional) |

See [PEER_SIGNAL.md](./PEER_SIGNAL.md) · [SDK media architecture](../../packages/niilox-sdk/docs/MEDIA_ARCHITECTURE.md)

---

## Frontends & repos

| Repo / path | Product | Backend usage | SDK usage |
|-------------|---------|---------------|-----------|
| `frontend/drift-web` | [driftin.live](https://driftin.live) livestream | Full LiveKit stack | None |
| `geogig/` | GeoGig mobile (Expo) | Auth, gigs, profile, verification, safety, P2P | **Partial** — video call, DMs, typing |
| `drift-admin/` → `Dr-M06/mango` | [admin.driftin.live](https://admin.driftin.live) | Ops, keys, docs, verifications UI | None |
| `locust-api-docs/` | Public GitHub docs | — | — |
| `packages/niilox-sdk` | Client library | Mirrors API | — |

### GeoGig app — migrated vs remaining

| Area | Uses SDK? | Still on `geogig/src/lib/api.ts` |
|------|-----------|----------------------------------|
| Video call + WebView P2P | Yes | — |
| DMs + typing | Yes | — |
| Auth (phone, Google, Apple, password) | No | Yes |
| Gigs, profile, verification, safety | No | Yes |
| Notifications, push | No | Yes |

### Admin portal (`drift-admin`)

| Item | Status |
|------|--------|
| Niilox rebrand (logo, copy, grouped ops nav) | Done |
| Developer docs + peer integration UI | Done |
| Verifications queue, announcements, push categories | Done |
| **Vercel production deploy** | Verify — may show old build if deploy failed |
| SDK install snippets in docs | Partial |

### Drift web

| Item | Status |
|------|--------|
| Native auth, rooms, gifts, VIP, DMs, typing | Done |
| Uses `@niilox/sdk` | No |

---

## SDK (`@niilox/sdk` v0.1)

Full detail: [`packages/niilox-sdk/README.md`](../../packages/niilox-sdk/README.md)

### TypeScript modules (24 on `NiiloxClient`)

| Module | API coverage |
|--------|----------------|
| `auth` | Guest, phone, magic, password (+ forgot/reset/set), Apple, Google init, refresh, logout, **admin MFA**, **WebAuthn** |
| `users` | Profile, avatar, socials, follow graph, discover, age confirm, spend-tokens, token balance |
| `rooms` | LiveKit — list, create, end, join, leave, presence, chat, VIP access, thumbnail |
| **`seats`** | **Capped events** — inventory, reserve, fiat checkout, lobby feed |
| `gifts` | Catalog, send, my gifters |
| `stage` | Co-host request/approve/deny/remove/mute/leave |
| `moderation` | Mute, unmute, kick |
| `payments` | Token packs, checkout, mobile config, fiat gift/room/gig/booking/identity, purchase poll |
| `wallet` | Summary, banks, payout, withdrawals |
| `gigs` | Full GeoGig lifecycle + `uploadImage`, `videoCall` |
| `profile` | GeoGig profile, mode, CV, trust, identity refund |
| `verification` | Catalog, submit, admin queue |
| `safety` | Settings, PINs, emergency contacts |
| `dms` | Conversations, thread, send, typing, media, delete, `PersonalSocket` |
| `notifications` | Feed, unread, mark read |
| `peer` | ICE, signal URL, `PeerSession`, devices, doorcam rings |
| `drop` | Mint, upload parts, unlock, cipher URL |
| `rodent` | Bookings, pairings, booking-settings, booking-page |
| `scheduled` | CRUD, start, remind subscribe/unsubscribe |
| `integrations` | Google Calendar connect/status/disconnect |
| `platform` | Ping, tenant, business, usage, keys, token-packs, ops, developer portal |
| `push` | Web push + native token register |
| `announcements` | Active, dismiss, admin CRUD |
| `rabbaly` | Feed, posts, PPV, tips |

Also exported: `RoomSocket`, `PersonalSocket`, `@niilox/sdk/react`, `@niilox/sdk/peer`. Seat and paywall helpers — integration guide on request.

### Python (`python/niilox/`)

Subset: auth, users, verification, profile, identity payments, wallet, DMs, peer URLs, gigs basics, **gifts, rooms, seats, stage, moderation, scheduled, rodent booking**. Not full parity with TypeScript.

### SDK gaps (intentional or backlog)

| Gap | Notes |
|-----|--------|
| **npm / PyPI publish** | Monorepo `file:` install only |
| **Integration tests + CI** | No automated suite against staging |
| **Auto token refresh** | `auth.refresh()` exists; no built-in interceptor (GeoGig still has `getValidToken()` in `api.ts`) |
| **Per-seat labels** (A12, B4) | Count-based caps only — see integration guide |
| **Stays / multi-night** | Not built (hotel/Airbnb needs date-range reservations) |
| `POST /push/send` | Server API-key only — use `client.request()` |
| `POST /peer/doorcam/session` | Device JWT (firmware), not app JWT |
| `GET /platform/push/categories` | Developer-portal push category admin |
| **OpenAPI** | Stub only (`openapi/openapi.yaml`) |
| **Dogfooding** | drift-web + most of GeoGig still bypass SDK |

---

## Migrations (database)

**59 SQL migrations** in `backend/drift/migrations/` (000–059).

| Migration | Feature |
|-----------|---------|
| 000–011 | Base schema, social, wallet, age gate, thumbnails |
| 012–019 | VIP rooms, payouts, API keys, native auth |
| 020–029 | Password auth, developers portal, tenant business |
| 030–040 | Scheduled streams, announcements, Google Calendar |
| 041–048 | Rodent bookings, **GeoGig gigs + payments** |
| 049–052 | GeoGig chef, categories, expiry |
| 053–054 | Push tokens, notification categories |
| 055–058 | GeoGig bios/CV, currency, **verification country** |
| 059 | **Worker safety** |

### Verify on production

| Migration | Notes |
|-----------|--------|
| `057_geogig_app_mobile_ids` | iOS bundle / Android package on `geogig` tenant |
| `059_worker_safety` | Apply if not yet on prod |

---

## What's left (prioritized)

### P0 — production correctness

| Task | Owner |
|------|-------|
| Confirm **admin.driftin.live** Vercel deploy (latest mango commit) | Ops |
| Deploy **BunnyFor / ensureGeogig** fix if ID upload still 503 on GeoGig | Backend |
| Apply migrations **057**, **059** if pending | Backend / DBA |
| Verify Flutterwave bank identity on production GeoGig | QA |

### P1 — product completeness

| Task | Notes |
|------|-------|
| Finish **GeoGig → SDK** migration (auth, gigs, profile, verification, safety) | App |
| **npm publish** `@niilox/sdk` | SDK |
| **Integration tests** — capped seats, gifts, paywall, webhooks | SDK + CI |
| SDK **token refresh** helper + migrate GeoGig `api.ts` | SDK + App |
| Sync **locust-api-docs** on doc changes (or automate) | Docs |
| Enable **media-svc** in prod (optional) | Ops |
| **Redis** when WS load needs second API replica | Ops |

### P2 — polish & scale

| Task | Notes |
|------|-------|
| Migrate **drift-web** to SDK (optional) | Frontend |
| Wire SDK snippets in **admin developer docs** | drift-admin |
| **OpenAPI** full spec + codegen | SDK |
| PyPI **`niilox`** package | SDK |
| iOS GeoGig prebuild on Mac | Mobile |
| Finland / EU bank-ID countries in verification catalog | Backend |
| Rodent static site booking UI (API already exists) | Rodent |
| Full **locust-api-docs** sync on every doc change | Docs |

### Known gaps (by design or backlog)

| Gap | Detail |
|-----|--------|
| No BVN/NIN API | Nigeria uses Flutterwave bank-transfer identity, not government ID API |
| P2P in Expo | WebView WebRTC for video glance; no `react-native-webrtc` yet |
| Supabase | Fully removed from production auth |
| Per-tenant Postgres schemas | Skipped — `app_id` column isolation used instead |

---

## Related documents

| Doc | Purpose |
|-----|---------|
| [API.md](./API.md) | Every endpoint |
| [GEOGIG.md](./GEOGIG.md) | GeoGig integration |
| [PEER_SIGNAL.md](./PEER_SIGNAL.md) | P2P / Rodent |
| [WORKER_SAFETY.md](./WORKER_SAFETY.md) | Safety PINs |
| [MIGRATION_STATUS.md](./MIGRATION_STATUS.md) | Supabase → native cutover |
| [PRODUCTION_RUNBOOK.md](./PRODUCTION_RUNBOOK.md) | Operator runbook |
| [WIRING_CHECKLIST.md](./WIRING_CHECKLIST.md) | Pre-launch checklist |
| [GEOGIG co-dev handoff](../../../docs/GEOGIG_CO_DEV_HANDOFF.md) | GeoGig session notes |
| [SDK README](../../packages/niilox-sdk/README.md) | Client library |

---

## Repos map

```
drift/                          monorepo root
├── backend/drift/              Go API + media-svc + migrations
├── frontend/drift-web/           Drift livestream (Next.js)
├── geogig/                       GeoGig Expo app
├── drift-admin/                  Niilox developer portal → GitHub mango
├── locust-api-docs/              Public docs mirror
├── packages/niilox-sdk/          TypeScript + Python SDK
└── docs/                         Cross-cutting handoffs
```

GitHub: `Dr-M06/locust-api` (docs) · `Dr-M06/geogig` · `Dr-M06/mango` (admin)
