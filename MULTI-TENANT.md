# Multi-tenancy

**Niilox API** hosts multiple branded apps (Drift, GeoGig, Rodent, Rabbaly, partner tenants) on one deployment. Tenants share:

- the binary,
- the Postgres database,
- the WebSocket hub,
- the (optional) Rust media-svc.

Tenants do **not** share:

- LiveKit projects (each tenant has its own host, key, secret),
- Bunny.net storage zones,
- payment provider credentials,
- adult-content policy (`apps.allow_adult`),
- user identities (the same Supabase account can have separate user rows per tenant).

## The `apps` table

Created by `migrations/008_apps.sql`:

```sql
CREATE TABLE apps (
  id                TEXT PRIMARY KEY,
  name              TEXT NOT NULL,
  livekit_host      TEXT NOT NULL DEFAULT '',
  livekit_key       TEXT NOT NULL DEFAULT '',
  livekit_secret    TEXT NOT NULL DEFAULT '',
  bunny_zone        TEXT NOT NULL DEFAULT '',
  bunny_key         TEXT NOT NULL DEFAULT '',
  bunny_cdn         TEXT NOT NULL DEFAULT '',
  bunny_region      TEXT NOT NULL DEFAULT '',
  payment_provider  TEXT NOT NULL DEFAULT 'stripe',
  allow_adult       BOOLEAN NOT NULL DEFAULT false,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

`apps.id` is the value that clients send in `X-App-ID`. The default tenant id is `drift` and is upserted automatically on boot from the env-var credentials (`LIVEKIT_*`, `BUNNY_*`).

## Adding a new tenant

```sql
INSERT INTO apps (
  id, name,
  livekit_host, livekit_key, livekit_secret,
  bunny_zone, bunny_key, bunny_cdn, bunny_region,
  payment_provider, allow_adult
) VALUES (
  'rabbaly',
  'Rabbaly',
  'wss://rabbaly.livekit.cloud',
  'APIxxxxxxxxxxxx',
  'secretxxxxxxxxx',
  'rabbaly-media',
  'storage-zone-password',
  'https://rabbaly.b-cdn.net',
  '',                        -- empty = EU
  'flutterwave',
  false
);
```

Restart the API (or wait until `apps.Registry.Load` is re-invoked on next deploy). Clients then call:

```
GET /api/v1/rooms
Authorization: Bearer <user_jwt>
X-App-ID: rabbaly
```

and get only Rabbaly rooms, Rabbaly LiveKit tokens, Rabbaly notifications.

## How the API knows the tenant

The decision happens once per request in `middleware.Tenant`:

```go
appID := r.Header.Get("X-App-ID")    // defaults to "drift"
app, ok := registry.Get(appID)        // 400 if unknown
ctx = context.WithValue(ctx, AppKey, app)
```

Handlers then call `middleware.GetApp(ctx)` or `middleware.GetAppID(ctx)`.

## LiveKit room names

`tenant.RoomName(appID, hostID)` builds `{appID}::{hostID}::{yyyymmddhhmmss}` so a LiveKit webhook can recover the tenant from the room name even though webhooks don't carry custom headers:

```go
func ParseRoomAppID(livekitRoom string) string {
    if i := strings.Index(livekitRoom, "::"); i > 0 {
        return livekitRoom[:i]
    }
    return "drift"
}
```

`webhook.LiveKitHandler.Handle`, `moderation.Manager.lkForRoom`, and `stage.Manager.lkForRoom` all use this to dispatch to the right tenant's LiveKit credentials.

## Per-tenant services

`apps.Registry` exposes:

```go
registry.LiveKit(app)  // *livekit.Service — caches one per app.ID
registry.Bunny(app)    // *bunny.Client    — caches one per app.ID
```

Both are lazy. The first call instantiates and caches; later calls hit the `sync.Map`.

Payments are handled by `payments.Registry`, which is process-wide (one `stripe` provider, one `flutterwave` provider) — the tenant only chooses **which** provider to use via `apps.payment_provider`.

## Data isolation

Every tenant-scoped table has an `app_id` column with a foreign key to `apps(id)`. Indexes are also (app_id, …):

| Table | Composite indexes |
|-------|-------------------|
| `users` | unique `(app_id, supabase_id)` |
| `rooms` | partial unique on active rooms by host, partial index on `(app_id, ended_at IS NULL)` |
| `chat_messages` | `(app_id, room_id, created_at DESC)` |
| `gift_transactions` | `(app_id, room_id, created_at DESC)` |
| `notifications` | `(app_id, user_id, created_at DESC)` and partial unread |
| `follows`, `direct_messages`, `withdrawal_requests`, `user_socials`, `id_verification_requests` | all scoped by `app_id` |

Every query in `internal/*/handler.go` filters by `app_id` — this is how the same Postgres database can host multiple distinct products without leakage.

## Adult-content gating

Three columns gate adult streams. Hosts and viewers see different gates:

- `apps.allow_adult` — tenant-wide toggle. Defaults to `false` on the seeded `drift` row. Flip it once with SQL **or** list the tenant id in the `ALLOW_ADULT_APPS` env var (e.g. `ALLOW_ADULT_APPS=drift`) — the server runs an idempotent `UPDATE apps SET allow_adult = true WHERE id = ANY($1)` on boot.
- `users.id_verified` — **hosts only**. Set by `/api/v1/admin/verifications/{id}/approve` after the host uploads a government ID + selfie.
- `users.age_confirmed_at` — **viewers**. Stamped by `POST /users/me/confirm-age` the first time the user accepts the in-app +18 modal. No paperwork; viewers just tick "I'm 18+".

Host gate — `POST /rooms`:

| `apps.allow_adult` | `is_adult` on body | Result |
|--------------------|--------------------|--------|
| `false` | `true` (or `category = adult`) | 403 `adult streams not allowed on this app` |
| `true`  | `true` and host `id_verified = false` | 403 `id verification required for adult streams` |
| `true`  | `true` and host `id_verified = true`  | created |

Viewer gate — `POST /rooms/{id}/join` on an adult room:

| Caller | `age_confirmed_at` | Result |
|--------|-------------------|--------|
| signed-in user | NULL  | 403 `age confirmation required to watch adult streams` |
| signed-in user | set   | joined |
| guest token    | n/a   | 403 `sign in to watch adult streams` |

The two host-side guards run in order — if the tenant flag is off you'll see the first message even on a verified account. Inspect the body of the 403, not just the status code, when you're debugging "but I'm verified" reports.

## Private / VIP rooms

Private rooms are tenant-agnostic — they layer on top of every existing gate without changing tenant isolation. The relevant columns live on `rooms`:

| Column | Meaning |
|--------|---------|
| `rooms.is_private` | When `true`, `POST /rooms/{id}/join` requires a row in `room_access` (host-bypassed). |
| `rooms.entry_price_tokens` | Per-viewer charge in tokens. `0` is valid ("free + tip-only"). Capped server-side at 10 000. |
| `rooms.seat_limit` | Max paid viewers; `0` means unlimited. Capped server-side at 500. |
| `room_access.(room_id, user_id)` | UNIQUE — atomic seat reservation + double-pay shortcut in one constraint. |

The `BuyAccess` handler is one transaction: `SELECT … FOR UPDATE` on the rooms row → seat-cap check → `UPDATE users SET token_balance = token_balance - price` (guarded by `>= price`, so 0 rows ⇒ paywall) → `UPDATE users SET token_balance += hostShare` for the host → `INSERT INTO room_access`. The host's cut uses the same `hostShareBps = 7000` (70 %) basis-points constant the gift split uses, so the platform's take is consistent across every revenue stream.

After commit the hub fans `lobby:seats` out to every signed-in user on `/ws/me`, so lobby cards and open paywall sheets tick "N of M seats left" downward without polling.

Adult ∩ Private is fully supported — both gates stack. A host creating a paid adult room still needs `apps.allow_adult = true` + `users.id_verified = true`; a viewer joining still needs `age_confirmed_at IS NOT NULL` **and** a paid `room_access` row.

## What can't be multi-tenant yet

The hub, stage manager, moderation manager are in-memory and **per-process**, but they keyed by `roomID` (a UUID from the `rooms` table, which is already tenant-unique). They work multi-tenant out of the box — you just don't get one hub per tenant, you get one hub across all tenants. No data leaks between rooms because each room belongs to exactly one tenant.

If you need stricter isolation (e.g. per-tenant rate limits, per-tenant Prometheus labels), wrap the hub or add middleware. The pattern is the same — pull `appID` off the context.
