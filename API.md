# API Reference

> **Public docs note:** Transaction semantics, paywall flows, worker safety, geofence logic, and seat-capacity implementation are withheld from this repository. Registered developers receive the full integration guide — contact **dev@niilox.com**.

**Niilox API** (`api.driftin.live`) exposes a single HTTP + WebSocket surface for all tenants (`drift`, `geogig`, `rodent`, and partner apps). Every request that mutates tenant-scoped data carries:

New to the API? Start with [`README.md`](./README.md) → [Getting started](./GETTING_STARTED.md). No-code: [Bubble.io](./BUBBLE_IO.md).

| Header | Required | Notes |
|--------|----------|-------|
| `Authorization: Bearer <drift_jwt>` | yes (most routes) | Issued by native auth routes (`/auth/magic/verify`, `/auth/phone/verify`, `/auth/apple`, `/auth/refresh`) or `/auth/guest`. Google OAuth redirects to the frontend with a one-time access token. Routes under `/api/v1` that do not write generally accept guest tokens; routes annotated **registered** below reject guests. |
| `X-App-ID` | optional | Selects the tenant. Defaults to `drift` when omitted. The header is read by the tenant middleware before auth and resolves to a row in the `apps` table. |
| `Content-Type` | yes for JSON bodies | `application/json` unless noted (avatar / DM media / ID docs use `multipart/form-data`). |

All paths below are relative to `/api/v1` unless prefixed with `/webhook…`, `/ws/…`, or `/health`. Errors are returned as `text/plain` with an appropriate HTTP status. Success bodies are JSON.

---

## Health

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/health` | — | Returns `{"ok":true}` (JSON). No `X-App-ID` or auth. Load-balancer / Bubble liveness. |

## Authentication

Production uses **native auth** (no Supabase). OAuth callback is outside `/api/v1`:

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/auth/google/callback` | — | Google OAuth redirect target. Redirects to `{APP_URL}/auth/finish?token=<access_jwt>`. |

| Method | Path | Auth | Body | Returns |
|--------|------|------|------|---------|
| POST | `/auth/google/init` | — | — | `{"url":"<google_oauth_url>"}` — start Google sign-in. |
| POST | `/auth/magic/send` | — | `{"email":"..."}` | `204` — sends magic link via Resend. |
| POST | `/auth/magic/verify` | — | `{"token":"..."}` | `{"token":"<access>", "refresh_token":"<refresh>", "user_id":"<uuid>"}` |
| POST | `/auth/phone/send` | — | `{"phone":"+234..."}` | `204` — sends OTP (when Africa's Talking configured). |
| POST | `/auth/phone/verify` | — | `{"phone":"+234...", "code":"123456"}` | Same shape as magic verify. |
| POST | `/auth/password/login` | — | `{"email":"...", "password":"..."}` | Same shape as magic verify. |
| POST | `/auth/password/register` | — | `{"email":"...", "password":"...", "name"?}` | Creates account + password (min 8 chars). `409` if password already set. |
| POST | `/auth/password/forgot` | — | `{"email":"...", "finish_url"?}` | `{"ok":true}` — emails a 15-minute reset link when the account has a password. Always returns `ok` (no email enumeration). Requires Resend. Mobile: `finish_url` e.g. `geogig://reset-password`. |
| POST | `/auth/password/reset` | — | `{"token":"...", "password":"..."}` | Same shape as magic verify — sets password and signs in. |
| POST | `/auth/password/set` | **JWT** | `{"password":"..."}` or `{"new_password":"...", "password":"<current>"}` | `204` — set or change password. |
| POST | `/auth/apple` | — | `{"identity_token":"..."}` | Same shape as magic verify. |
| POST | `/auth/refresh` | — | `{"refresh_token":"..."}` | New access + refresh pair. |
| POST | `/auth/logout` | — | `{"refresh_token":"..."}` (optional) | `204` — revokes refresh session. |
| POST | `/auth/guest` | — | — | `{"token": "<guest_jwt>", "user_id": "guest-...", "name": "guest-XXXX"}` — 24-hour watch-only JWT. Guests can list / view rooms; write routes reject them. |

Access tokens are HS256 JWTs (7-day TTL). Refresh tokens are opaque (30-day TTL, stored hashed in `auth_sessions`).

> **Legacy:** `POST /auth/verify` (Supabase JWKS bridge) was removed from production. See [NATIVE_AUTH.md](./NATIVE_AUTH.md).

## Users

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/users/me` | yes | Current user profile + follower / following counts + social links. Rodent tenant also returns `rodent_pro_active`, `rodent_pro_until`, and `booking_price_cents` (host default session price in USD cents). |
| PUT | `/users/me` | yes | Update display name (2–24 chars) and avatar URL. |
| PUT | `/users/me/socials` | yes | Replace whitelisted social links (`nomli`, `rabbaly`, `linktree`). |
| POST | `/users/me/avatar` | yes | `multipart/form-data`, field `avatar`. Accepts JPEG / PNG / WebP up to 5 MB. Returns `{"avatar_url": "..."}`. |

## Rooms

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/rooms` | optional | List active rooms for the current tenant. Optional `?category=fun|music|sports|gossip|vibe|adult`. |
| GET | `/rooms/{roomID}` | optional | Single room metadata with live viewer count. |
| POST | `/rooms` | **registered** | Body: `{"title":"...","category":"vibe","is_adult":false,"is_private":false,"entry_price_tokens":0,"seat_limit":0}`. Creates one active room per host. Adult rooms require `apps.allow_adult = true` **and** the host to have `users.id_verified = true`. Private (VIP) rooms accept tokens at the door — see **Private VIP rooms** below. Returns LiveKit token + room name. |
| DELETE | `/rooms/active` | **registered** | Ends the caller's currently active room. |
| DELETE | `/rooms/{roomID}` | **registered** | Ends a specific room (only owner). |
| POST | `/rooms/{roomID}/join` | yes | Issues a viewer LiveKit token. Adult rooms reject guests and unverified users. Private rooms return **402** with the paywall snapshot if the caller hasn't paid (host is always exempt). |
| POST | `/rooms/{roomID}/leave` | yes | 204 (the client side closes the LiveKit connection). |
| GET | `/rooms/{roomID}/presence` | yes | `{count, viewers:[{user_id, username, avatar_url}]}` snapshot. |
| GET | `/rooms/{roomID}/access` | yes | VIP paywall snapshot — see **Private VIP rooms** below. Safe on public rooms (returns `has_access=true`, `is_private=false`). |
| POST | `/rooms/{roomID}/access` | **registered** | Grants VIP access (token spend). Returns paywall snapshot on success or when payment is required. Idempotent for callers who already paid. Contact **dev@niilox.com** for full flow documentation. |
| POST | `/rooms/{roomID}/thumbnail` | **registered, host-only** | Raw JPEG body (`Content-Type: image/jpeg`, max 300 kb). The host's browser captures a frame from the local camera every ~30 s and POSTs it; the API uploads to Bunny at `thumbnails/{room_id}.jpg` (per-tenant zone) and updates `rooms.thumbnail_url` with a cache-busted CDN URL. Returns `{thumbnail_url}`. Lobby list responses include the latest `thumbnail_url` so cards can render a real preview instead of an avatar gradient. |

### Private VIP rooms

Paid VIP rooms support per-viewer token pricing and optional seat caps. Hosts set `is_private`, `entry_price_tokens`, and `seat_limit` at go-live.

Access endpoints return a paywall snapshot (`has_access`, `entry_price_tokens`, `seat_limit`, `seats_taken`, `balance`, etc.). Live lobby counters update over the personal WebSocket.

**Implementation details** (transaction semantics, paywall status codes, and realtime events) are provided to registered developers — contact **dev@niilox.com**.

## Chat

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/rooms/{roomID}/chat` | — | Most-recent 100 messages oldest-first. |
| POST | `/rooms/{roomID}/chat` | **registered** | Body: `{"text":"..."}` (≤300 chars). Rejected with 403 if the moderator has the user muted in this room. Broadcasts `chat` WS event. |
| DELETE | `/rooms/{roomID}/chat/{msgID}` | **registered** | Removes a chat line the **caller authored** (sender-only — moderation-level deletion is a separate route). 404 on any non-owner or missing id. Broadcasts `chat:deleted`. |

## Gifts & token balance

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/gifts` | — | Static catalog (id, name, emoji, token cost). |
| POST | `/rooms/{roomID}/gifts` | **registered** | Body: `{"gift_id":"heart"}`. Deducts tokens, credits the host, broadcasts a `gift` event. Returns `{gift, balance}`. |
| GET | `/users/me/tokens` | **registered** | `{"balance": int}`. |

## Stage (guest co-hosts)

A room caps at 4 publishers (1 host + 3 guests).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/rooms/{roomID}/stage` | yes | `{roster, requests, capacity, filled}` snapshot. |
| POST | `/rooms/{roomID}/stage/request` | **registered** | Viewer requests to go on stage. |
| DELETE | `/rooms/{roomID}/stage/request` | **registered** | Viewer cancels their own request. |
| POST | `/rooms/{roomID}/stage/approve/{userID}` | **registered, host-only** | Promotes a requester. Grants `CanPublish=true` on LiveKit. |
| POST | `/rooms/{roomID}/stage/deny/{userID}` | **registered, host-only** | Rejects a request. |
| POST | `/rooms/{roomID}/stage/remove/{userID}` | **registered, host or self** | Drops a guest publisher (revokes publish rights). |
| POST | `/rooms/{roomID}/stage/mute/{userID}` | **registered, host-only** | Body: `{"muted":true|false}`. Server-side mute (LiveKit Cloud disables remote unmute — the API still returns 204 and emits `stage:muted` so the guest's client can unmute locally). |
| POST | `/rooms/{roomID}/stage/leave` | **registered** | Caller leaves the stage. |

## Moderation

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/rooms/{roomID}/moderation` | yes | `{muted: [...], banned: [...]}` with expiry timestamps. |
| POST | `/rooms/{roomID}/moderation/mute/{userID}` | **registered** | Mutes the target's chat for 10 minutes. |
| DELETE | `/rooms/{roomID}/moderation/mute/{userID}` | **registered** | Clears the mute. |
| POST | `/rooms/{roomID}/moderation/kick/{userID}` | **registered** | Kicks the target from LiveKit + WS; blocks rejoin for 15 minutes. |

## Social

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/users/{userID}/profile` | yes | Public profile + follower / following counts + `is_following` + `is_online` + socials. |
| POST | `/users/{userID}/follow` | yes | Idempotent. Notifies the followee (deduped 24 h). |
| DELETE | `/users/{userID}/follow` | yes | Unfollow. |
| GET | `/users/{userID}/followers` | yes | Up to 500 entries with online + live-now badges. |
| GET | `/users/{userID}/following` | yes | Same shape as `/followers`. |

## Direct messages

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/dms` | yes | Conversations ordered by latest message. |
| GET | `/dms/{userID}` | yes | Last 100 messages with this peer. |
| POST | `/dms/{userID}` | yes | Body: `{"text":"...","media_url":"..."}` (≤500 chars; media must be a Bunny URL we issued). |
| DELETE | `/dms/{messageID}` | yes | Removes a DM the **caller sent** (sender-only). `{messageID}` is the DM's own primary key, not the peer user id. Pushes `dm:deleted` on `/ws/me` to both sender and recipient so the message disappears across every open device. 404 on any non-owner or missing id. |
| POST | `/dms/{userID}/typing` | yes | Fires `dm:typing` to the recipient's personal channel. Fire-and-forget. |
| POST | `/dms/media` | yes | `multipart/form-data`, field `file`. JPEG / PNG / WebP / GIF ≤8 MB. Returns `{"media_url":"..."}`. |

## Notifications

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/notifications` | yes | Paginated feed (`?limit=1..200`, default 50) + `unread_count`. |
| GET | `/notifications/unread_count` | yes | `{count: int}` only. |
| POST | `/notifications/read` | yes | Body: `{"all":true}` or `{"ids":["uuid", ...]}`. |

## Push

Web (PWA) and native (iOS/Android) device registration. Sending (`POST /push/send`) requires a tenant API key — not an end-user JWT.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/push/public-key` | — | VAPID public key for web push subscribe (empty when VAPID not configured). |
| POST | `/me/push/subscribe` | yes | Body: Web Push `PushSubscription` JSON — stores browser endpoint for the caller. |
| POST | `/me/push/unsubscribe` | yes | Body: `{"endpoint":"..."}` — removes a web subscription. |
| POST | `/me/push-tokens` | yes | Body: `{"platform":"ios\|android","token":"...","device_id":"..."}` — registers native APNs/FCM token (`getDevicePushTokenAsync`, not Expo push tokens). |
| DELETE | `/me/push-tokens` | yes | Body: `{"platform":"ios\|android","token":"..."}` — unregisters a native token (e.g. on sign-out). |
| POST | `/push/send` | **api key** | Server-triggered push to a user (integrator / ops). |
| GET | `/platform/push/categories` | **developer portal** | List push notification categories for the tenant. |
| POST | `/platform/push/categories` | **developer portal** | Create or update a category slug + default payload template. |

SDK: `client.push` (subscribe, native token register). Category admin is portal-only today.

## Platform (B2B API keys)

Tenant integrators authenticate with `Authorization: Bearer drift_sk_…` plus `X-App-ID`. When `hmac_required` is true on the key, also send `X-Drift-Timestamp` (unix seconds) and `X-Drift-Signature` (hex HMAC-SHA256 of `{ts}\n{METHOD}\n{path}\n{sha256(body)}`).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/platform/ping` | **api key or user JWT** | Health check for integrators (`{ok, app_id, auth, key_id?}`). |
| GET | `/platform/tenant` | **admin JWT** | Current tenant summary (`X-App-ID`): users, live rooms, active keys. |
| GET | `/platform/tenants` | **admin JWT** (`X-App-ID: drift` only) | List all B2B tenants (ops). |
| POST | `/platform/tenants` | **admin JWT** (`X-App-ID: drift` only) | Body: `{"id","name"}` — provision tenant (copies infra from `drift`). |
| GET | `/platform/usage?days=30` | **admin JWT** | Usage stats for billing (users, rooms, chat, gifts, keys). |
| GET | `/platform/usage/timeseries?days=30` | **admin JWT** | Daily usage trend for charts (`new_users`, `rooms_started`, `chat_messages`, `token_purchases`, `gift_tokens`). |
| GET | `/platform/usage/geo?days=30` | **admin JWT** | API request geo split by country code (`CF-IPCountry`) with request, 4xx, and 5xx totals. |
| GET | `/platform/keys` | **admin JWT** | List API keys for the tenant (secrets never returned). |
| POST | `/platform/keys` | **admin JWT** | Body: `{"name","scopes":["read"],"hmac_required":false}`. Returns the full `secret` once. |
| DELETE | `/platform/keys/{keyID}` | **admin JWT** | Revoke a key. |
| POST | `/platform/keys/{keyID}/rotate` | **admin JWT** | Revoke + mint replacement; returns new `secret` once. |
| POST | `/platform/test-sign` | **admin JWT** | Body: `{"secret","method","path","body"}` → `{signature, headers, curl_example}` for the dashboard tester. |

**Admin UI:** `https://admin.driftin.live` — separate app in repo `drift-admin/`. Partner devs: grant `users.is_admin` on their `app_id`, then sign in on the admin portal with `?app=<tenant>`.

All `/api/v1` routes are rate-limited per IP and per `X-App-ID` (Redis when `REDIS_URL` is set). **Video routes** (`POST /rooms/{id}/join`, go-live, LiveKit tokens) still require a normal user JWT — API keys do not replace end-user sessions.

## Wallet & withdrawals

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/users/me/wallet` | **registered** | `{summary, activity[…]}` with `withdrawable = earned_from_gifts - withdrawn - pending`. Both `spent_on_gifts` and `earned_from_gifts` are inclusive: they tally every creator-economy debit / credit (gifts **and** VIP-room entries). The activity feed adds `vip_paid` (viewer) and `vip_earned` (host) item kinds for each `room_access` row. |
| GET | `/banks` | — | Non-exhaustive global bank suggestions for the payout typeahead: `[{code, name}, …]`. Optional `?q=` substring filter (case-insensitive). |
| GET | `/users/me/payout-details` | **registered** | Saved bank account or `{}` if unset. Includes `complete` when bank name, account number (4–34 chars), and account name are present. |
| PUT | `/users/me/payout-details` | **registered** | Body: `{"bank_name","bank_code"?,"account_number","account_name"}`. `bank_name` is free-text (typed or chosen from suggestions); `bank_code` is auto-filled from the catalog when omitted and `bank_name` matches a known suggestion. |
| POST | `/users/me/withdrawals` | **registered** | Body: `{"tokens": int >=100}`. Atomically deducts from balance, creates a `pending` withdrawal request. Withdrawable amount is capped at gift earnings. |

## Token packs & checkout

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/tokens/packs` | — | Static catalog. Each pack includes `store_product_id` for App Store / Play (mobile ignores on web). |
| GET | `/tokens/mobile-config` | — | Mobile only: `{ provider: "revenuecat", revenuecat_api_key_apple, revenuecat_api_key_google, packs }`. Use with `react-native-purchases`; do not use `/tokens/checkout` on native. |
| POST | `/tokens/checkout` | **registered** | **Web only.** Body: `{"pack_id":"pack_150","return":"/lobby/all"}`. Stripe/Flutterwave checkout URL. Mobile must use RevenueCat IAP instead. |
| POST | `/payments/checkout` | **registered** | Alias of `/tokens/checkout` — provider-neutral name. |
| POST | `/payments/fiat/booking` | **registered** | **Rodent only.** Body: `{"booking_id":"<uuid>","return":"/path","finish_url":"https://rodent.nomli.cc"}`. Starts Stripe Checkout for a paid session booking (`scheduled_streams` row with `payment_status=pending`). Returns `{"checkout_url":"...","order_id":"...","amount_cents":N}` or `{"already_paid":true}`. Host share **97%** (`BookingHostShareBps = 9700`). Requires Stripe on the `rodent` tenant. |
| POST | `/payments/fiat/gig` | **registered** | **Geo-Gig only.** Body: `{"gig_id":"<uuid>","provider":"stripe\|flutterwave","finish_url":"geogig://","return":"/gig/{id}"}`. Poster pays gig `price_cents` via Stripe or Flutterwave. Returns `{"checkout_url","order_id","amount_cents","provider"}` or `{"already_paid":true}`. Worker is paid (90% host share) on check-out. Migration: `048_geogig_payments.sql`. |

## Scheduled sessions

Calendar slots for Rodent (and other tenants that use scheduling). Rows live in `scheduled_streams`. List responses include booking fields when set: `booker_id`, `price_cents`, `duration_min`, `payment_status`, `google_synced`, `remind_subscribed`.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/scheduled` | optional | Upcoming slots. `?mine=true` limits to the caller as host or booker (**registered**). |
| GET | `/scheduled/{id}` | optional | Single slot. |
| POST | `/scheduled` | **registered** | Host creates a slot. Body: `{"title":"...","category":"session","starts_at":"<RFC3339>","timezone":"Europe/Helsinki"}`. |
| PATCH | `/scheduled/{id}` | **registered** | Host updates title, time, or status (`scheduled` / `cancelled` / `ended`). |
| DELETE | `/scheduled/{id}` | **registered** | Host deletes a future slot. |
| POST | `/scheduled/{id}/start` | **registered** | Host marks slot live and mints a peer room code (`room_id` on the row). |
| POST | `/scheduled/{id}/remind` | **registered** | Subscribe email reminder for this slot. |
| DELETE | `/scheduled/{id}/remind` | **registered** | Unsubscribe reminder. |

Paid bookings created via `POST /rodent/bookings` are also `scheduled_streams` rows (see below).

## Geo-Gig gigs (`X-App-ID: geogig`)

Local chore/help listings for the [GeoGig](https://github.com/Dr-M06/geogig) mobile app. Integration guide: [GEOGIG.md](./GEOGIG.md). Migrations: `042`–`048` (gigs, sessions, reviews, lifecycle, payments).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/gigs/categories` | — | Category catalog for the tenant. |
| GET | `/gigs` | — | Open gigs. `?category=`, `?sort=price_asc\|price_desc\|distance\|starts_at`, `?lat=&lng=&radius_m=` optional. |
| GET | `/gigs/{id}` | — | Single gig (includes `payment_status`: `none\|pending\|paid\|released`). |
| GET | `/gigs/mine` | **registered** | Caller’s posted or accepted gigs. `?role=posted\|accepted\|all`, `?view=active\|history`, `?status=` optional. |
| POST | `/gigs` | **registered** | Post a gig. Requires poster identity verification. Body includes category, title, summary, price, optional location fields, and scheduling. |
| PUT | `/gigs/{id}` | **registered** | Poster edits an open gig. |
| DELETE | `/gigs/{id}` | **registered** | Poster cancels/deletes when allowed by lifecycle rules. |
| POST | `/gigs/{id}/accept` | **registered** | Accept an open gig (cannot accept your own). Requires worker identity verification. |
| POST | `/gigs/{id}/cancel` | **registered** | Cancel by poster or worker per lifecycle rules. |
| POST | `/gigs/{id}/no-show` | **registered** | Report no-show. |
| POST | `/gigs/{id}/dispute` | **registered** | Open a dispute on an active/completed gig. |
| POST | `/gigs/{id}/image` | **registered** | `multipart/form-data` gig photo (JPEG/PNG/WebP). |
| POST | `/gigs/{id}/video-call` | **registered** | Ring the other party for P2P video glance (uses peer signaling, not LiveKit). |
| GET | `/gigs/{id}/session` | **registered** | Live session state (check-in/out, location trust). |
| GET | `/gigs/{id}/location-trail` | **registered** | Worker location history for the session (poster view). |
| POST | `/gigs/{id}/location` | **registered** | Worker GPS ping while gig is accepted. Rate-limited. |
| POST | `/gigs/{id}/check-in` | **registered** | Worker manual check-in. Requires paid status when price applies. Optional body `{"lat","lng"}`. |
| POST | `/gigs/{id}/check-out` | **registered** | Worker check-out. Optional body `{"lat"?,"lng"?,"pin"?}`. See **Worker safety** below. |
| GET | `/gigs/{id}/review-pending` | **registered** | Whether caller can review the other party; includes reviewee reputation snapshot. |
| POST | `/gigs/{id}/review` | **registered** | Body `{"stars":1-5,"comment"?}`. One review per user per gig. |
| GET | `/gigs/users/{userId}/rating` | — | `{"user_id","average","count"}` aggregate for Geo-Gig reviews received. |
| GET | `/gigs/users/{userId}/reviews` | — | Recent reviews for a user (`?limit=20`). |

### GeoGig profile & trust

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/profile/me` | **registered** | GeoGig profile, verification country, bank lock state. |
| PUT | `/profile/me` | **registered** | Update bio, skills, availability fields. |
| PUT | `/profile/mode` | **registered** | Body `{"mode":"worker\|poster"}`. |
| PUT | `/profile/verification-country` | **registered** | Set country for ID verification. |
| POST | `/profile/cv` | **registered** | Upload CV (`multipart/form-data`). |
| POST | `/profile/identity/refund-request` | **registered** | Request identity verification fee refund. |
| GET | `/profile/trust/{userId}` | **registered** | Public trust snapshot for another user. |

**Realtime** (personal WebSocket `wss://…/ws/me?token=<jwt>`):

| Event | Payload | When |
|-------|---------|------|
| `gig:checked_in` | `{gig_id, worker_id, lat?, lng?}` | Worker checks in |
| `gig:checked_out` | same | Worker checks out |
| `gig:location` | same | Worker location ping |
| `gig:geofence_alert` | same | Location trust alert |

Bell notification kinds for the poster include check-in, check-out, and location alerts (`data.gig_id`, `data.title`).

Auth uses standard native endpoints — Google with `portal: "geogig"`, phone OTP (`/auth/phone/*`), password, Apple. Web Google finish URL: `{GEOGIG_APP_URL}/auth/finish?token=…`.

## Worker safety (all tenants; GeoGig first)

Field-worker safety for gig apps — checkout verification, emergency contacts, and location trust.

Public documentation intentionally omits implementation detail. Registered developers receive the full integration guide — contact **dev@niilox.com**. See [WORKER_SAFETY.md](./WORKER_SAFETY.md).

## Rodent paid bookings (`X-App-ID: rodent`)

Host-to-client session booking with optional card checkout. Media stays **peer-to-peer** (WebRTC); the API only stores metadata and payment state. Migration: `041_rodent_bookings.sql`.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/rodent/hosts/{hostID}/booking-page` | — | Public host card + occupied future slots for a booking UI. Returns `{host:{id,display_name,avatar_url,booking_price_cents},occupied:[{starts_at,duration_min}]}`. |
| GET | `/rodent/booking-settings` | yes | Caller’s default session price: `{price_cents}`. |
| PUT | `/rodent/booking-settings` | yes | Body: `{"price_cents":N}` (0 = free, max 500000). **Rodent Pro** required when `price_cents > 0`. |
| POST | `/rodent/bookings` | **registered** | Book a slot with a host. Body: `{"host_id":"<uuid>","starts_at":"<RFC3339>","timezone":"...","title":"Session","duration_min":60}` (duration 15–480, default 60). Returns `{id,payment_required,amount_cents,payment_status}` where `payment_status` is `waived` (free) or `pending` (paid). **409** if the slot overlaps an existing booking. |

**Checkout flow (paid):**

1. Client calls `POST /rodent/bookings` → `payment_required: true`, `payment_status: pending`.
2. Client calls `POST /payments/fiat/booking` with `booking_id` → redirect to Stripe Checkout.
3. Stripe webhook (`commerce_kind=booking`) marks the row `payment_status=paid` and credits the host’s token balance (97% of gross, same basis-points model as other commerce).

Integrators can build their own booking UI against these routes; the static Rodent site may omit booking UI while the API remains available.

## Google Calendar (Rodent hosts)

Optional OAuth sync for `scheduled_streams` slots. Requires `GOOGLE_CALENDAR_*` env on the API host. OAuth callback: `GET /auth/google/calendar/callback` (redirects to the tenant app with `?gcal=connected`).

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/integrations/google/calendar/status` | yes | `{connected,email?}` |
| POST | `/integrations/google/calendar/connect` | yes | Returns `{url}` — redirect the user to Google consent. |
| DELETE | `/integrations/google/calendar` | yes | Disconnect and stop sync. |

New/updated/deleted Rodent slots push to the connected calendar automatically when sync is enabled.

## Peer signaling & Drop-off (Rodent)

WebRTC signaling and encrypted async file handoff for the `rodent` tenant. Not duplicated here — see [`PEER_SIGNAL.md`](./PEER_SIGNAL.md) (`GET /peer/ice`, `GET /peer/signal`, `POST /drop/mint`, …).

## ID verification

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/verification/me` | **registered** | `{id_verified, verified_at, status}` (status is the latest submission). |
| POST | `/verification/submit` | **registered** | `multipart/form-data` with two fields: `doc` and `selfie`. Each ≤8 MB. Files upload to Bunny under `verify/{user_id}/`. |
| GET | `/admin/verifications` | **admin** | Pending queue (oldest first). |
| POST | `/admin/verifications/{id}/approve` | **admin** | Marks `users.id_verified = true`. |
| POST | `/admin/verifications/{id}/reject` | **admin** | Rejects without flipping the verified flag. |

`adminOnly` is satisfied when `users.is_admin = true` for the calling user in the current tenant.

## Webhooks

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/webhook/livekit` | LiveKit auth-header HMAC | Ingests LiveKit Cloud webhooks. `room_finished` ends the matching DB row. `track_published` starts a quality watch in the Rust media-svc when enabled. The handler infers `app_id` from the LiveKit room name (`{app}::{host}::{ts}`). |
| POST | `/webhooks/stripe` | Stripe-Signature | Credits tokens on `checkout.session.completed`. Idempotent: only updates rows still in `pending`. |
| POST | `/webhooks/revenuecat` | `Authorization: Bearer <REVENUECAT_WEBHOOK_AUTH>` | **Mobile only.** Credits tokens on `INITIAL_PURCHASE` / `NON_RENEWING_PURCHASE`; clawback on `REFUND`. See [MOBILE_PAYMENTS.md](./MOBILE_PAYMENTS.md). |
| POST | `/webhooks/payments/{provider}` | provider-specific | Generic provider webhook. Today routes to Stripe (`stripe`) or Flutterwave (`flutterwave`). Adding a new provider only requires implementing `payments.Provider`. |

## WebSockets

```
GET /ws/rooms/{roomID}?token=<drift_jwt>
GET /ws/me?token=<drift_jwt>
```

Both are server → client only. Clients should reconnect with exponential backoff. The server sends WebSocket pings every 30 s and closes the connection after 60 s without a pong.

### Room channel events (`/ws/rooms/{roomID}`)

| Event | Payload |
|-------|---------|
| `chat` | `Message` (see `/rooms/{id}/chat`) |
| `chat:deleted` | `{id, room_id, user_id}` — sender deleted one of their own lines |
| `gift` | `{gift, sender_id, username}` |
| `room:seats` | `{seats_taken, seat_limit}` — VIP room only. Fires whenever a new viewer pays for access. |
| `join` | `{user_id, username}` |
| `leave` | `{user_id, username}` |
| `presence` | `{count, viewers:[{user_id, username, avatar_url}]}` |
| `viewers` | reserved (currently same shape as presence) |
| `end` | `{room_id}` |
| `room:quality_warn` | `{identity, kind, detail}` — emitted by the Rust quality cop (warn) or by the API when it mutes/kicks a publisher for stream quality. |
| `stage:requested` | `Member` |
| `stage:cancelled` | `{user_id}` |
| `stage:approved` | `Member` |
| `stage:removed` | `Member` |
| `stage:roster` | `{roster, requests, capacity, filled}` |
| `stage:muted` | `{user_id, muted}` |
| `moderation:muted` | `{user_id, username, expires_at}` |
| `moderation:unmuted` | `{user_id}` |
| `moderation:kicked` | `{user_id, expires_at}` |

### Personal channel events (`/ws/me`)

| Event | Payload |
|-------|---------|
| `dm` | `DMMessage` |
| `dm:deleted` | `{id, sender_id, recipient_id}` — sender removed a DM; pushed to both parties |
| `dm:typing` | `{from}` |
| `user:online` | `{user_id}` |
| `user:offline` | `{user_id}` |
| `notification` | `Notification` |
| `notification:unread` | `{count}` |
| `lobby:seats` | `{room_id, seats_taken, seat_limit}` — VIP room seat counter pushed to every signed-in user so lobby cards update in real time. |

Online / offline events fire only for the peer set returned by `social.PeersToNotify` (DM history ∪ follow graph), capped at 500 peers per transition.

## Limits & defaults

| Constraint | Value |
|------------|-------|
| Chat message length | 300 chars |
| DM text length | 500 chars |
| DM media size | 8 MB |
| Avatar size | 5 MB |
| ID verification doc / selfie | 8 MB each (JPEG/PNG/WebP/PDF) |
| Stage capacity | 4 publishers (1 host + 3 guests) |
| Withdrawal minimum | 100 tokens |
| VIP entry price | 0 – 10 000 tokens |
| VIP seat limit | 0 (unlimited) – 500 |
| Live notification dedupe | 30 min (live), 24 h (follow) |
| Followers fan-out cap on go-live | 5000 |
