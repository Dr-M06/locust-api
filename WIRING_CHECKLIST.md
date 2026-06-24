# Integration wiring checklist

Use this before going to production. Check each box in order.

## Account & tenant

- [ ] Developer account at https://admin.driftin.live  
- [ ] Tenant `app_id` chosen (lowercase, e.g. `mybrand`)  
- [ ] `X-App-ID: <app_id>` documented for your team  

## Credentials

- [ ] At least one **API key** (`drift_sk_…`) created and stored server-side only  
- [ ] `/platform/ping` succeeds with key + `X-App-ID`  
- [ ] API key rotation process documented (create new → deploy → revoke old)  

## End-user auth

Pick one or more:

- [ ] Magic link (`POST /auth/magic/send` → user clicks link → `POST /auth/magic/verify`)  
- [ ] Google (`POST /auth/google/init` with `portal: "developer"` or your app flow)  
- [ ] Email + password (`/auth/password/register`, `/auth/password/login`)  
- [ ] Guest mode for browse-only (`POST /auth/guest`)  

Storage:

- [ ] Access JWT stored securely (httpOnly cookie or secure storage on mobile)  
- [ ] Refresh token stored and `POST /auth/refresh` on 401  
- [ ] Logout calls `POST /auth/logout` when you store refresh tokens  

## Core product flows

- [ ] `GET /users/me` after login  
- [ ] `GET /rooms` lobby  
- [ ] `POST /rooms` (host go-live) — registered users only  
- [ ] `POST /rooms/{id}/join` + LiveKit client in your UI  
- [ ] `GET/POST /rooms/{id}/chat`  
- [ ] `POST /rooms/{id}/gifts` or fiat gift checkout if using fiat mode  
- [ ] VIP: `GET/POST /rooms/{id}/access` or `POST /payments/fiat/room-access`  

## Realtime

- [ ] WebSocket reconnect with backoff  
- [ ] Handle `chat`, `gift`, `presence`, `end` on room socket  
- [ ] Handle `notification`, `lobby:seats` on `/ws/me` if using VIP lobby  

## Payments

- [ ] [Tenant business](./TENANT_BUSINESS.md) configured (split, mode, packs)  
- [ ] Stripe Connect `acct_…` if you collect card payments  
- [ ] Web: `POST /tokens/checkout` or `POST /payments/checkout`  
- [ ] Fiat VIP/gifts: `POST /payments/fiat/room-access`, `POST /payments/fiat/gift`  
- [ ] Mobile: RevenueCat — [MOBILE_PAYMENTS.md](./MOBILE_PAYMENTS.md)  

## Security

- [ ] Never embed `drift_sk_…` in frontend or Bubble **page** workflows visible to users  
- [ ] CORS: your web origin allowed (contact support if custom domain)  
- [ ] Webhook signature verification if you proxy Stripe events  

## Bubble.io (if applicable)

- [ ] [Bubble guide](./BUBBLE_IO.md) completed  
- [ ] API Connector uses **backend workflows** for secrets  
- [ ] Dynamic `Authorization` header uses **user JWT**, not API key  

## Go-live

- [ ] Staging tenant tested end-to-end  
- [ ] Error handling for 402 (paywall), 403 (banned/suspended), 409 (room full)  
- [ ] Monitoring on `GET /health` and your own API error rates  
