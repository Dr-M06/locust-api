# GeoGig integration (`X-App-ID: geogig`)

[GeoGig](https://github.com/Dr-M06/geogig) is the reference mobile app for local gigs — chores, dog walks, errands, and help around the neighborhood. It runs on **Niilox API** at `https://api.driftin.live/api/v1` with tenant header `X-App-ID: geogig`.

## What GeoGig uses from Niilox

| Area | Transport | Notes |
|------|-----------|-------|
| Auth | REST | Google (web + native), Apple, password, magic link, **phone SMS OTP** |
| Gigs & sessions | REST + `/ws/me` | Post, accept, check-in/out, reviews |
| Payments | REST + webhooks | Fiat gig checkout |
| DMs & social | REST + `/ws/me` | Peer chat between poster and worker |
| P2P video glance | Rodent peer API | P2P video for gigs — **not LiveKit** |
| Worker safety | REST | Field-worker safety — contact **dev@niilox.com** |
| Identity & trust | REST | Bank-verified display name, CV upload, verification country |
| Push | Native + web | `POST/DELETE /me/push-tokens` (APNs/FCM); web VAPID via `/me/push/subscribe` — SDK `push` |

LiveKit rooms (`/rooms/*`) are **not** used by GeoGig.

## Auth

All auth routes are standard Niilox native auth. Always send `X-App-ID: geogig`.

### Google (web)

```http
POST /api/v1/auth/google/init
Content-Type: application/json

{"portal":"geogig"}
```

Opens Google OAuth. Callback redirects to `GEOGIG_APP_URL/auth/finish?token=<jwt>` (configure `GEOGIG_APP_URL` on the API host).

### Google (mobile)

```json
{"portal":"geogig","finish_url":"geogig://auth"}
```

### Phone SMS OTP

Requires Africa's Talking on the API host (`AFRICASTALKING_USERNAME`, `AFRICASTALKING_API_KEY`).

```bash
curl -s https://api.driftin.live/api/v1/auth/phone/send \
  -H "X-App-ID: geogig" \
  -H "Content-Type: application/json" \
  -d '{"phone":"+2348012345678"}'

curl -s https://api.driftin.live/api/v1/auth/phone/verify \
  -H "X-App-ID: geogig" \
  -H "Content-Type: application/json" \
  -d '{"phone":"+2348012345678","code":"123456"}'
```

Returns `{token, refresh_token, user_id}` — same shape as magic link verify.

### Password reset (mobile deep link)

```json
{"email":"user@example.com","finish_url":"geogig://reset-password"}
```

## Gigs flow

1. **Poster** completes identity verification → posts a gig.
2. **Worker** verifies identity → accepts.
3. **Poster** pays when price applies → fiat checkout.
4. **Worker** checks in and completes the gig.
5. Both parties review.

List open gigs: `GET /gigs` with optional category, sort, and location filters.

Categories: `GET /gigs/categories`.

Full lifecycle documentation: **dev@niilox.com**.

## Realtime (`/ws/me`)

Gig events (check-in, check-out, location, alerts), DMs, and bell notifications are pushed on the personal WebSocket.

Connect: `wss://api.driftin.live/ws/me?token=<access_jwt>`.

## P2P video glance

GeoGig uses **peer signaling**, not LiveKit. Use the `gigs` and `peer` SDK modules.

See [PEER_SIGNAL.md](./PEER_SIGNAL.md). Contact **dev@niilox.com** for full examples.

## Push (native)

GeoGig registers **native APNs/FCM device tokens** (not Expo push tokens) after sign-in:

```http
POST /api/v1/me/push-tokens
Authorization: Bearer <jwt>
X-App-ID: geogig

{"platform":"ios|android","token":"<device_token>","device_id":"ios"}
```

```http
DELETE /api/v1/me/push-tokens
{"platform":"ios|android","token":"<device_token>"}
```

Requires `APNS_*` and/or `FCM_SERVICE_ACCOUNT_*` on the API host. Notification tap deep links are handled in the app (`geogig://` / Expo Router).

**Not yet:** server fan-out of high-priority `gig_available` pushes when a new gig is posted (workers still rely on in-app WS + manual refresh for fastest-finger).

## Profile & trust (`/profile/*`)

GeoGig-specific profile routes (require auth, `X-App-ID: geogig`):

| Method | Path | Description |
|--------|------|-------------|
| GET | `/profile/me` | Worker/poster profile, bank lock, verification state |
| PUT | `/profile/me` | Update bio, skills, availability |
| PUT | `/profile/mode` | `worker` or `poster` mode |
| PUT | `/profile/verification-country` | Country for ID verification flow |
| POST | `/profile/cv` | Upload CV (multipart) |
| POST | `/profile/identity/refund-request` | Request identity verification fee refund |
| GET | `/profile/trust/{userId}` | Public trust snapshot for a user |

Bank-verified users display their **bank account name** on profile (not editable display name).

## Worker safety

See [WORKER_SAFETY.md](./WORKER_SAFETY.md). Full integration guide: **dev@niilox.com**.

## Admin & ops

- **Developer portal:** https://admin.driftin.live (`/portal`, tenant `geogig`)
- **Ops:** verifications queue, announcements, push categories — grouped under `/ops`
- **Env on API host:** `GEOGIG_APP_URL`, `CORS_EXTRA_ORIGINS` (web app origin), Africa's Talking keys

## Reference client

Source: https://github.com/Dr-M06/geogig  
API header: `X-App-ID: geogig`

Full endpoint list: [API.md](./API.md#geo-gig-gigs-x-app-id-geogig).
