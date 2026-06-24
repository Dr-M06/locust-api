# Drift platform — worker safety

Reusable safety layer for gig apps (GeoGig first). Tenants opt in by mounting `/safety/*` routes and binding checkout in their session handler.

## Why this exists

A normal "I'm safe" checkout can be coerced. Worker safety adds:

1. **Missed-checkout timeout** — checked in, never checked out after gig end + grace → SMS emergency contacts.
2. **Duress PIN** — same success UI as normal checkout; server silently alerts contacts and keeps location tracking.

## API (`X-App-ID` + auth)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/safety/me` | Settings + emergency contacts |
| PUT | `/api/v1/safety/me` | `{ "enabled", "checkout_grace_minutes" }` (5–240, default 30) |
| PUT | `/api/v1/safety/me/pins` | `{ "checkout_pin", "duress_pin" }` — 4–8 digits, bcrypt hashed |
| POST | `/api/v1/safety/me/contacts` | `{ "name", "phone_e164", "relationship?", "sort_order?" }` |
| PUT | `/api/v1/safety/me/contacts/{id}` | Update contact |
| DELETE | `/api/v1/safety/me/contacts/{id}` | Remove contact |

## Gig checkout integration (GeoGig)

`POST /api/v1/gigs/{id}/check-out` accepts optional `pin` in the JSON body.

- No PINs configured → checkout works as before (backward compatible).
- PINs configured → `pin` required.
- Normal PIN → standard checkout + payment release + poster notification.
- Duress PIN → **same response** as normal; server additionally:
  - Sets `gig_sessions.duress_at`, `tracking_active = true`
  - SMS + in-app alert to emergency contacts
  - Continues accepting `POST /gigs/{id}/location` pings while app shows completed

## Database (migration `059_worker_safety.sql`)

- `worker_safety_settings` — per user per app
- `worker_emergency_contacts` — E.164 phones
- `worker_safety_alerts` — audit log (`duress_checkout`, `missed_checkout`)
- `gig_sessions` — `duress_at`, `tracking_active`, `missed_checkout_alert_at`

## Background worker

`StartMissedCheckoutWorker` runs every 2 minutes:

- Gig `accepted`, checked in, not checked out
- Past `ends_at_ts + grace` OR (no end time) `checked_in + 8h + grace`
- Alerts contacts once per session (`missed_checkout_alert_at`)

## SMS

Uses Africa's Talking (`AFRICASTALKING_USERNAME`, `AFRICASTALKING_API_KEY`). If unset, alerts are logged + in-app notification only.

## Adding to a new tenant app

1. Run migration `059_worker_safety.sql`.
2. Wire `workersafety.New` + `Handler.Routes` in `cmd/server/main.go`.
3. In your session checkout handler, call `ClassifyCheckoutPIN` and `OnDuressCheckout`.
4. Allow location pings when `AllowsSilentTracking` is true.
5. Build profile UI against `/safety/me`.

## Non-goals (v1)

- Hold-duration / triple-tap triggers
- Police dispatch integration
- Visible duress UI on device

## Legal copy (product)

> Not a replacement for emergency services. If you are in immediate danger, call your local emergency number.

## Code

- Package: `internal/workersafety/`
- GeoGig binding: `internal/geogig/session.go` (`check-out`, `location`)
- Notification kinds: `worker_safety_duress`, `worker_safety_missed_checkout`
