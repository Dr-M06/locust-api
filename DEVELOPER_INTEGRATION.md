# Developer Integration Guide

## What Drift is

Drift (`driftin.live`) is a live streaming and creator-economy platform built for low-bandwidth markets. It supports web and mobile clients, keeps sessions resilient on weak networks, and gives creators realtime monetization tools (gifts, VIP rooms, wallet, withdrawals) out of the box.

For an external developer, the key point is simple: one API integration gives you livestream rooms, realtime chat, stage/co-host controls, identity gates, and monetization without stitching multiple systems yourself.

Drift gives you a production-ready live-streaming + creator-economy backend so you can ship in days instead of building rooms, chat, gifts, VIP access, wallet, moderation, identity, and realtime plumbing from scratch. One integration covers web and mobile clients, supports low-bandwidth networks, and includes multi-tenant isolation for B2B use cases.

If you are choosing between assembling your own stack (or wiring multiple vendors), this API is the faster path when you want one contract for livestream operations and monetization without exposing infrastructure secrets.

This guide is for partner developers integrating quickly and safely. It intentionally avoids operator-only secrets and internal infrastructure details.

Use this together with the full reference in [`API.md`](./API.md).

**New docs hub:** [`README.md`](./README.md) · [Getting started](./GETTING_STARTED.md) · [Wiring checklist](./WIRING_CHECKLIST.md) · [Bubble.io](./BUBBLE_IO.md) · [Payments & revenue](./TENANT_BUSINESS.md)

## 1) Base URL and auth model

- Base API URL (production): `https://api.driftin.live/api/v1`
- Tenant selector: `X-App-ID: <tenant_id>` (defaults to `drift` if omitted)
- Auth types:
  - End-user JWT (`Authorization: Bearer <drift_jwt>`) for user flows (rooms, chat, gifts, DMs, wallet)
  - Tenant API key (`Authorization: Bearer drift_sk_...`) for platform/B2B checks (`/platform/ping`)

Important: API keys are not a replacement for end-user session tokens.

## 2) Minimal integration checklist

1. Choose your tenant id (for example `locust`).
2. Create or receive a tenant API key (`drift_sk_...`).
3. Set `X-App-ID` on every request from your app/backend.
4. Implement user sign-in using native auth endpoints.
5. Connect WebSocket channels for realtime updates.
6. Add retries for idempotent GET requests and reconnect WS with backoff.

## 3) First calls (copy/paste)

### Health check

```bash
curl -s https://api.driftin.live/health
```

### Platform ping (B2B API key)

```bash
curl -s https://api.driftin.live/api/v1/platform/ping \
  -H "Authorization: Bearer drift_sk_your_key_here" \
  -H "X-App-ID: locust"
```

Expected shape:

```json
{"ok":true,"app_id":"locust","auth":"api_key"}
```

### Guest token (watch-only session)

```bash
curl -s https://api.driftin.live/api/v1/auth/guest \
  -H "Content-Type: application/json" \
  -H "X-App-ID: locust" \
  -d "{}"
```

### List live rooms

```bash
curl -s "https://api.driftin.live/api/v1/rooms?category=music" \
  -H "Authorization: Bearer <guest_or_user_jwt>" \
  -H "X-App-ID: locust"
```

## 4) Typical user flow (recommended)

1. Sign in via Google, magic link, phone, Apple, or password.
2. Call `GET /users/me` to hydrate profile state.
3. Show lobby from `GET /rooms`.
4. Join a room via `POST /rooms/{roomID}/join`.
5. Open WebSocket:
   - room events: `/ws/rooms/{roomID}?token=<jwt>`
   - personal events: `/ws/me?token=<jwt>`

For private VIP rooms:

- Check access with `GET /rooms/{roomID}/access`
- Unlock seat with `POST /rooms/{roomID}/access`
- On 402, show top-up flow (`/tokens/checkout` for web, RevenueCat for mobile)

### TypeScript SDK (optional)

For web, React, or Expo apps in the monorepo, use **`@niilox/sdk`** (`packages/niilox-sdk/`) instead of hand-rolling fetch + WebSocket wiring:

- **24 modules** on `NiiloxClient` — auth, rooms, **seats** (capped VIP/events), gifts, stage, moderation, payments, gigs, peer, DMs, push, platform, …
- Realtime: `PersonalSocket`, `RoomSocket`
- **v0.1 beta** — monorepo `file:` install only; not on npm yet

```ts
import { createNiiloxClient } from '@niilox/sdk'

const client = createNiiloxClient({ appId: 'myapp', token: jwt })
const rooms = await client.rooms.list()
// VIP / seats — contact dev@niilox.com for integration guide
```

Full module list and install: [SDK README](../../packages/niilox-sdk/README.md) · [Platform status](./PLATFORM_STATUS.md#sdk-niiloxsdk-v01)

## 5) Realtime events you should handle first

Room channel (`/ws/rooms/{roomID}`):

- `chat`
- `gift`
- `presence`
- `room:seats` (VIP seat count — integration guide on request)
- `end`

Personal channel (`/ws/me`):

- `notification`
- `notification:unread`
- `dm`
- `dm:typing`
- `lobby:seats` (VIP lobby updates — integration guide on request)

## 6) Security and privacy guardrails

Do:

- Keep API keys and JWT refresh tokens server-side where possible.
- Rotate compromised keys immediately.
- Verify webhook signatures (Stripe / provider webhooks).
- Use `X-App-ID` consistently to preserve tenant isolation.

Do not:

- Expose operator `.env` values in frontend bundles.
- Hardcode production secrets in mobile apps or public repos.
- Use platform API keys to perform end-user media actions.

## 7) Example apps you can build on this API

- Live fan community app (rooms + chat + gifts + follows)
- Creator coaching rooms with paid VIP seats
- Audio-first social spaces for low-bandwidth regions
- B2B hosted streaming product for agencies/brands (multi-tenant)
- Event companion app with realtime chat and private backstage rooms
- University clubs / campus live network with moderated stages

## 8) Integration notes by client type

Web app:

- Use JWT + `/tokens/checkout` for token top-ups.
- Reconnect WebSockets with exponential backoff and jitter.

Mobile app:

- Use RevenueCat (`GET /tokens/mobile-config` + RC SDK) for token packs.
- Do not open web checkout in native apps.

Backend-to-backend:

- Use `drift_sk_...` keys for platform checks (`/platform/ping`).
- If your key requires HMAC, include `X-Drift-Timestamp` and `X-Drift-Signature`.

## 9) Where to go next

- Docs index: [`README.md`](./README.md)
- Step-by-step first integration: [`GETTING_STARTED.md`](./GETTING_STARTED.md)
- Pre-launch checklist: [`WIRING_CHECKLIST.md`](./WIRING_CHECKLIST.md)
- **Bubble.io (no-code):** [`BUBBLE_IO.md`](./BUBBLE_IO.md)
- Tenant payments & splits: [`TENANT_BUSINESS.md`](./TENANT_BUSINESS.md)
- Full endpoint reference: [`API.md`](./API.md)
- **SDK:** [`packages/niilox-sdk/README.md`](../../packages/niilox-sdk/README.md) (monorepo)
- Native auth details: [`NATIVE_AUTH.md`](./NATIVE_AUTH.md)
- Mobile token purchases: [`MOBILE_PAYMENTS.md`](./MOBILE_PAYMENTS.md)
- Multi-tenant behavior: [`MULTI-TENANT.md`](./MULTI-TENANT.md)

## 10) Suggested first partner conversation

Use this sequence:

1. **Problem framing:** "You need livestream + monetization + moderation, but don't want to build infra."
2. **What Drift gives immediately:** rooms, stage, chat, gifts, VIP door payments, wallet, verification, realtime events.
3. **Proof in production:** `driftin.live` is live on the same backend contract.
4. **Integration path:** run `/platform/ping`, issue app key, wire auth, then launch one room flow.
5. **Pilot scope:** one tenant + one client app + one monetization path, then expand.

For database migrations and production ops backlog, see [Platform status](./PLATFORM_STATUS.md) (59 migrations; verify **057** and **059** on prod).
