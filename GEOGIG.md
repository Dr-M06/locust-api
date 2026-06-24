# GeoGig integration (`X-App-ID: geogig`)

[GeoGig](https://github.com/Dr-M06/geogig) is the reference mobile app for local gigs — chores, dog walks, errands, and help around the neighborhood. It runs on **Niilox API** at `https://api.driftin.live/api/v1` with tenant header `X-App-ID: geogig`.

## What GeoGig uses from Niilox

| Area | Transport | Notes |
|------|-----------|-------|
| Auth | REST | Google (web + native), Apple, password, magic link, **phone SMS OTP** |
| Gigs & sessions | REST + `/ws/me` | Post, accept, geofence check-in/out, reviews |
| Payments | REST + webhooks | Stripe / Flutterwave gig checkout (`POST /payments/fiat/gig`) |
| DMs & social | REST + `/ws/me` | Peer chat between poster and worker |
| P2P video glance | Rodent peer API | `POST /gigs/{id}/video-call` + `/peer/ice` — **not LiveKit** |
| Worker safety | REST | Checkout PIN, duress PIN, emergency contacts — see [WORKER_SAFETY.md](./WORKER_SAFETY.md) |
| Identity & trust | REST | Bank-verified display name, CV upload, verification country |
| Push | Native routes | Expo push tokens via `/push/native/*` |

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

1. **Poster** completes identity verification → `POST /gigs` with category, price, optional geofence (`lat`, `lng`, `geofence_radius_m`).
2. **Worker** verifies identity → `POST /gigs/{id}/accept`.
3. **Poster** pays (when `price_cents ≥ 50`) → `POST /payments/fiat/gig` → redirect to Stripe/Flutterwave.
4. **Worker** checks in (`POST /gigs/{id}/check-in`) — manual or automatic via `POST /gigs/{id}/location` inside geofence.
5. **Worker** checks out (`POST /gigs/{id}/check-out`) — optional `pin` when worker safety PINs are set.
6. Both parties review via `POST /gigs/{id}/review`.

List open gigs: `GET /gigs?category=&sort=price_asc|price_desc|distance|starts_at&lat=&lng=&radius_m=`.

Categories: `GET /gigs/categories`.

## Realtime (`/ws/me`)

| Event | When |
|-------|------|
| `gig:checked_in` | Worker enters geofence or taps check-in |
| `gig:checked_out` | Worker leaves geofence or taps check-out |
| `gig:location` | Worker GPS ping |
| `gig:geofence_alert` | Dog walk exceeds `walk_radius_m` |
| `dm` / `dm:typing` | Direct messages |
| `notification` | Bell feed (check-in, safety alerts, etc.) |

Connect: `wss://api.driftin.live/ws/me?token=<access_jwt>`.

## P2P video glance

GeoGig uses **Rodent-style peer signaling**, not LiveKit:

1. `GET /peer/ice` with `X-App-ID: geogig` → STUN/TURN servers.
2. `POST /gigs/{id}/video-call` → notifies the other party (bell + WS).
3. WebSocket `wss://api.driftin.live/api/v1/peer/signal?app_id=geogig` for SDP/ICE exchange.

See [PEER_SIGNAL.md](./PEER_SIGNAL.md).

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

See [WORKER_SAFETY.md](./WORKER_SAFETY.md). Summary:

- `GET/PUT /safety/me` — enable safety, grace minutes
- `PUT /safety/me/pins` — checkout + duress PINs
- `POST/PUT/DELETE /safety/me/contacts` — emergency contacts (SMS via Africa's Talking)
- `POST /gigs/{id}/check-out` with `{"pin":"..."}` when PINs are configured

## Admin & ops

- **Developer portal:** https://admin.driftin.live (`/portal`, tenant `geogig`)
- **Ops:** verifications queue, announcements, push categories — grouped under `/ops`
- **Env on API host:** `GEOGIG_APP_URL`, `CORS_EXTRA_ORIGINS` (web app origin), Africa's Talking keys

## Reference client

Source: https://github.com/Dr-M06/geogig  
API header: `X-App-ID: geogig`

Full endpoint list: [API.md](./API.md#geo-gig-gigs-x-app-id-geogig).
