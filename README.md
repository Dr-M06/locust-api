# Niilox API — developer documentation

**Multi-tenant platform API** for live video, local gigs, peer signaling, authentication, and billing.

| | |
|---|---|
| **API** | `https://api.driftin.live/api/v1` |
| **Health** | `https://api.driftin.live/health` |
| **Developer portal** | [admin.driftin.live](https://admin.driftin.live) — keys, billing, usage, in-browser docs |
| **Status** | [status.driftin.live](https://status.driftin.live) |

Every request needs header **`X-App-ID: <your_tenant>`**. User actions use a session JWT; server integrations use **`drift_sk_…`** API keys.

---

## Quick start

**1. Create a tenant** at [admin.driftin.live](https://admin.driftin.live) → sign in → **Create account** → pick an app id (e.g. `myapp`).

**2. Ping the API** with your key:

```bash
curl -s https://api.driftin.live/health
# {"ok":true}

curl -s https://api.driftin.live/api/v1/platform/ping \
  -H "Authorization: Bearer drift_sk_YOUR_KEY" \
  -H "X-App-ID: myapp"
```

**3. Sign in a test user** (guest, 24 h):

```bash
curl -s https://api.driftin.live/api/v1/auth/guest \
  -H "X-App-ID: myapp" \
  -H "Content-Type: application/json" \
  -d "{}"
```

Full walkthrough → [**Getting started**](./GETTING_STARTED.md) (~15 min).

---

## Documentation

### Start here

| Guide | For |
|-------|-----|
| [**Getting started**](./GETTING_STARTED.md) | First tenant, first API call |
| [**Wiring checklist**](./WIRING_CHECKLIST.md) | End-to-end launch checklist |
| [**Developer integration**](./DEVELOPER_INTEGRATION.md) | Day-to-day flows, WebSockets |
| [**API reference**](./API.md) | Every endpoint |

### Auth & users

| Guide | For |
|-------|-----|
| [**Native auth**](./NATIVE_AUTH.md) | Google, Apple, magic link, **phone SMS OTP**, passwords |
| [**Authentication overview**](./AUTHENTICATION.md) | JWT model, guest tokens, refresh |

### Products on Niilox

| Guide | Tenant | Stack |
|-------|--------|-------|
| [**GeoGig**](./GEOGIG.md) | `geogig` | Gigs, geofence, fiat checkout, P2P video, worker safety |
| [**Peer signaling**](./PEER_SIGNAL.md) | `rodent`, `geogig` | WebRTC ICE + signal — **not LiveKit** |
| [**Worker safety**](./WORKER_SAFETY.md) | `geogig` (+ others) | Checkout PIN, duress PIN, emergency SMS |
| Live video, chat, gifts | `drift` | See [API reference](./API.md) — rooms, LiveKit, VIP |

Reference apps: [GeoGig](https://github.com/Dr-M06/geogig) · [Drift](https://driftin.live)

### Payments & no-code

| Guide | For |
|-------|-----|
| [**Payments**](./PAYMENTS.md) | Token packs, Stripe / Flutterwave, webhooks |
| [**Mobile payments**](./MOBILE_PAYMENTS.md) | RevenueCat / App Store / Play |
| [**Bubble.io**](./BUBBLE_IO.md) | No-code API Connector setup |

### Platform

| Guide | For |
|-------|-----|
| [**Multi-tenant**](./MULTI-TENANT.md) | `X-App-ID`, isolation, provisioning |
| [**Security**](./SECURITY.md) | Auth model, tenant isolation, payments |

---

## First-party tenants

| `X-App-ID` | Product |
|------------|---------|
| `drift` | Drift — live streaming, chat, gifts, VIP rooms |
| `geogig` | GeoGig — local gigs, safety, P2P video glance |
| `rodent` | Rodent — peer sessions, Drop, paid bookings |
| `rabbaly` | Rabbaly |
| *yours* | Provision via the [developer portal](https://admin.driftin.live) |

---

## Credentials (do not mix)

| Credential | Header | Used for |
|------------|--------|----------|
| **Tenant API key** | `Authorization: Bearer drift_sk_…` | Server / Bubble backend — `/platform/*` |
| **User session JWT** | `Authorization: Bearer <access_token>` | End users — rooms, gigs, chat, wallet |

---

## About this repo

This repository is the **public documentation mirror** for integrators. It contains guides only — no server source code or production secrets.

- **In-browser docs:** [admin.driftin.live/portal/dashboard/docs](https://admin.driftin.live/portal/dashboard/docs)
- **About Niilox:** named after [Niilo](https://admin.driftin.live/about) — see the portal about page

Questions or access requests: sign in at [admin.driftin.live](https://admin.driftin.live) or open an issue on this repo.
