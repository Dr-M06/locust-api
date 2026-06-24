# Peer signaling API (Rodent / WebRTC)

Tenant-scoped **WebRTC signaling** (SDP, ICE, room join) and **STUN/TURN** for `RTCPeerConnection`. Encrypted chat and files stay **peer-to-peer** on data channels.

## ICE / STUN / TURN (required for browsers)

Clients must load ICE from the API — **do not** ship TURN passwords in static `config.json` for production.

```bash
curl -s "https://api.driftin.live/api/v1/peer/ice" \
  -H "X-App-ID: rodent"
```

Response:

```json
{
  "app_id": "rodent",
  "ice_servers": [
    { "urls": "stun:stun.l.google.com:19302" },
    { "urls": "turn:turn.example.com:3478", "username": "…", "credential": "…" }
  ],
  "turn_configured": true,
  "signaling_path": "/api/v1/peer/signal"
}
```

Pass `ice_servers` directly to `new RTCPeerConnection({ iceServers })`.

### Server configuration

| Variable | Purpose |
|----------|---------|
| `PEER_TURN_URLS` | Comma-separated `turn:` / `turns:` URIs |
| `PEER_TURN_USERNAME` + `PEER_TURN_CREDENTIAL` | Static TURN login |
| `PEER_TURN_AUTH_SECRET` | Coturn `use-auth-secret` — API mints **time-limited** creds per tenant (`expiry:app_id`) |
| `PEER_TURN_CREDENTIAL_TTL_SEC` | Lifetime for minted creds (default 86400) |
| `PEER_STUN_URLS` | Optional STUN list (defaults to Google public STUN) |
| `PEER_OPEN_RELAY_FALLBACK` | If `true` and no `PEER_TURN_URLS`, returns Metered Open Relay (dev only) |

**CORS:** add your app origin to `CORS_EXTRA_ORIGINS` (e.g. `https://rodent.nomli.cc`) so browser `fetch('/peer/ice')` succeeds.

## Signaling WebSocket

| Method | Path | Tenant |
|--------|------|--------|
| GET | `/api/v1/peer/signal?app_id=TENANT` | Query required in browsers |
| GET | `/api/v1/peer/signal` | Or header `X-App-ID` |

Optional: `?token=<locust_jwt>`.

Protocol matches the legacy Rodent Node server wire format (`join`, `offer`, `answer`, `ice`, `peer-arrived`, `peer-left`). Rooms are isolated per tenant.

## Example (third-party app)

```javascript
const APP_ID = 'my_app'
const API = 'https://api.driftin.live/api/v1'

const { ice_servers } = await fetch(`${API}/peer/ice`, {
  headers: { 'X-App-ID': APP_ID }
}).then(r => r.json())

const pc = new RTCPeerConnection({ iceServers: ice_servers })

const ws = new WebSocket(
  `wss://api.driftin.live/api/v1/peer/signal?app_id=${APP_ID}`
)
```

## Other endpoints

| GET | `/api/v1/peer/health` | Tenant-scoped status |

## Drop-off (Rodent Pro)

Encrypted file links for signed-in Pro users. Uses the **same** `BUNNY_STORAGE_*` env as avatars/DM media — no separate Rodent storage zone.

| Method | Path | Auth |
|--------|------|------|
| POST | `/api/v1/drop/mint` | User JWT + `X-App-ID: rodent` (Pro required) |
| PUT | `/api/v1/drop/{id}/part` | Upload token from mint |
| POST | `/api/v1/drop/{id}/complete` | Upload token |
| GET | `/api/v1/drop/{id}/info` | Public |
| POST | `/api/v1/drop/{id}/unlock` | Optional PIN |
| GET | `/api/v1/drop/{id}/cipher?token=` | Download token; supports Range |

Set `DROP_ENABLED=false` to disable. Expired drops are purged by a background worker (`DROP_CLEANUP_INTERVAL`, default 15m).

## Paid session bookings

Rodent hosts can take **paid session bookings** (card checkout via Stripe) while media still runs peer-to-peer in the room. API routes (`X-App-ID: rodent`):

| Method | Path | Notes |
|--------|------|-------|
| GET | `/api/v1/rodent/hosts/{hostID}/booking-page` | Public — host profile + occupied slots |
| GET/PUT | `/api/v1/rodent/booking-settings` | Host default `price_cents` (Pro if &gt; 0) |
| POST | `/api/v1/rodent/bookings` | **Registered** — creates `scheduled_streams` row |
| POST | `/api/v1/payments/fiat/booking` | **Registered** — Stripe Checkout; 97% host share |

Calendar slots and Google Calendar OAuth: `/api/v1/scheduled`, `/api/v1/integrations/google/calendar/*`. See [`API.md`](./API.md).

## Disable

`PEER_SIGNAL_ENABLED=false` — Drift live rooms (`/ws/rooms`, LiveKit) unchanged.
