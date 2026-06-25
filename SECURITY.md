# Security model

This document is for anyone running Drift in production or considering reselling access to the API. It covers what the codebase enforces today, what it explicitly does not, and the safe defaults to add if you go to market.

## What the code enforces

### Authentication & authorisation

- All `/api/v1/*` routes that touch tenant data require either a registered Drift JWT or a guest JWT (both HS256 signed with `JWT_SECRET`).
- `middleware.RequireRegistered` rejects guest tokens with 403 on every write route (room creation, chat, gifts, payments, DMs, withdrawals, stage actions, moderation, ID verification).
- Native auth providers issue Drift access + refresh tokens; refresh sessions are stored hashed in `auth_sessions`.
- Per-tenant LiveKit tokens are minted with the tenant's own LiveKit secret (`apps.livekit_secret`). One tenant cannot mint tokens for another.
- Admin endpoints (`/admin/verifications/*`) check `users.is_admin` for the calling user in the current tenant on every call. There is no `is_admin` claim in the JWT.

### Tenant isolation

- Every tenant-scoped query carries `app_id`. There is no implicit cross-tenant access path.
- LiveKit webhooks recover `app_id` from the LiveKit room name (`{app}::{host}::{ts}`) and verify the webhook signature against that tenant's LiveKit secret.
- `apps.Registry` caches per-tenant LiveKit and Bunny clients keyed by `apps.id`.
- `users` remains tenant-scoped. Identities are isolated by `app_id` and mapped to tenant-local user rows.

### Input handling

- Display names are trimmed and capped at 24 chars.
- Chat messages are capped at 300 chars.
- DM text is capped at 500 chars; media URLs must originate from the tenant's Bunny CDN (`bunny.IsOwnURL`).
- Avatar / DM media / ID-verification uploads are size-limited (5 MB / 8 MB / 8 MB each).
- Social link handles are sanitised: trimmed, length-capped (200), and `javascript:` / `data:` / `vbscript:` URLs are rejected.
- ID verification files are named with kind + timestamp (`verify/{user}/doc-20260521131500.jpg`) so `doc` and `selfie` don't collide.
- Withdrawal amounts must be ≥100 tokens and cannot exceed the user's `earned_from_gifts - paid - pending` balance (i.e. you cannot withdraw money you topped up).

### Payment safety

- `payments.handler.handleWebhook` updates `token_purchases` with `WHERE status='pending'` so a replayed webhook never double-credits.
- The Stripe handler verifies `Stripe-Signature` against `STRIPE_WEBHOOK_SECRET`.
- The Flutterwave handler verifies `verif-hash` against `FLW_WEBHOOK_HASH` (raw value or SHA-512 hex digest).
- Token balance updates run inside a transaction when withdrawing.

### Chargeback / friendly fraud (migration `015`)

Drift listens for four Stripe lifecycle events and resolves them through the same handler:

| Event | Effect |
|-------|--------|
| `checkout.session.completed` | Credit tokens. Stores `payment_intent` on `token_purchases` so future dispute / refund webhooks can find the row. |
| `charge.dispute.created` | Soft clawback. Tokens revoked, `users.dispute_count++`. After two events the account is auto-suspended (`suspended_at` set, `suspension_reason='chargeback_threshold'`). |
| `charge.dispute.closed` (`status=lost`) | Permanent clawback. Marks purchase `disputed`. |
| `charge.dispute.closed` (`status=won` / `warning_closed`) | Decrements `dispute_count` and lifts the suspension if it was caused by chargebacks. Tokens are **not** restored — the user already lost them on `dispute_open`. |
| `charge.refunded` | Same as `dispute_lost`: clawback + flag, purchase marked `refunded`. |

Idempotency: every dispute write hits `payment_disputes` first, which has `UNIQUE (provider, external_id, kind)`. A replayed Stripe event is a no-op.

`RequireActive(db)` middleware blocks suspended users on `/tokens/checkout` and `/rooms/{id}/gifts` (the two laundering surfaces). The middleware caches the lookup for 5 s; `InvalidateSuspensionCache(userID)` forces an immediate re-read after an admin unsuspends.

| `payment_intent.payment_failed` | Marks the `token_purchases` row `failed` (card declined). No credit. |

**Webhook destination configuration.** Subscribe your Stripe endpoint at `https://api.driftin.live/webhooks/stripe` to all six events above. Until you do, dispute clawbacks will silently no-op (the events never reach the server).

### Realtime hub

- Each WS connection authenticates via the `?token=` query param parsed against `JWT_SECRET`.
- Slow clients are dropped from broadcasts (channel buffers of 64 / 128). They don't back-pressure the hub.
- `DisconnectUser` closes every connection a kicked user has in a room.

## What the code does NOT enforce

These are intentional gaps you should close before selling the API publicly.

| Concern | Today | Recommended hardening |
|---------|-------|------------------------|
| Per-IP / per-app rate limiting | **yes** — Redis sliding window on `/api/v1` (in-memory fallback without `REDIS_URL`) | Tune `RATE_LIMIT_*_PER_MIN`; add WAF for edge DDoS. Per-user limits apply when JWT is present on the same request. |
| Tenant API keys + HMAC | **yes** — `drift_sk_…` keys in `app_api_keys`; optional `X-Drift-Timestamp` + `X-Drift-Signature` | Manage keys at `/admin/api` (UI) or `GET/POST /api/v1/platform/keys`. Video/join routes still require user JWT — keys are for integrator read/ping flows. |
| Orphan room reconcile | **yes** — background worker compares LiveKit `ListRooms` vs DB | Tune `RECONCILE_INTERVAL`; does not delete LiveKit rooms. |
| PgBouncer / pool tuning | **partial** — `DB_POOL_MAX_CONNS` / `DB_POOL_MIN_CONNS` on pgxpool | Deploy PgBouncer in transaction mode in front of Postgres (see `DEPLOYMENT.md`). |
| WebSocket origin checks | `CheckOrigin` returns `true` | Replace with an allowlist (`APP_URL` + any partner origins). |
| Brute-force protection on auth routes | partial | Add per-IP limits / captcha on `/auth/password/*`, `/auth/magic/send`, and `/auth/phone/send`. |
| Login throttling / lockout | partial | Add progressive lockout policy for repeated password failures. |
| Server-side webhook IP filtering | none | Only accept Stripe / Flutterwave / LiveKit webhooks from their documented IP ranges or via signed payload only (the latter is already enforced). |
| Audit log | none | Add an `audit_log` table for admin actions (verifications, withdrawals, kicks). |
| Soft-delete of users | none — `ON DELETE CASCADE` removes follows, DMs, etc. | Add a `deleted_at` column and switch foreign keys to `ON DELETE SET NULL` if you need data retention. |
| Encryption at rest of DM media | files are public CDN URLs | Use Bunny's token-based signed URLs; rewrite `bunny.Upload` to issue signed reads. |
| Anti-CSRF | the API is JWT-bearer only, not cookie-based, so CSRF is not directly applicable | Keep using `Authorization: Bearer`. Never store the Drift JWT in cookies. |

## Secrets

- `JWT_SECRET` — 32+ random bytes. Rotating invalidates all sessions; users re-sign in.
- `LIVEKIT_SECRET` — per tenant in `apps.livekit_secret`. Don't read it from env in handlers — always go through `apps.Registry`.
- `STRIPE_KEY`, `STRIPE_WEBHOOK_SECRET`, `FLW_SECRET_KEY`, `FLW_WEBHOOK_HASH` — process-scoped. Rotate via your secrets manager.
- `BUNNY_STORAGE_KEY` — writable. Limit blast radius by giving the storage zone its own credentials (Bunny supports per-zone passwords).
- `.env` is git-ignored. The repo refuses to commit any `.env*` except `.env.example`.

## Threat model snapshot

| Adversary | What they can try | What stops them |
|-----------|-------------------|-----------------|
| Random internet user | Hit `/api/v1/*` without auth | `middleware.Auth` returns 401. |
| Guest token holder | Send chat, buy tokens, create rooms | `RequireRegistered` returns 403. |
| Cross-tenant attacker | Send `X-App-ID: rabbaly` while signed into Drift | The Drift JWT's `sub` is `users.id` for `app_id='drift'`. The Rabbaly handlers SELECT with `WHERE app_id='rabbaly' AND id=$sub` and get nothing. |
| Replayed webhook | Resubmit a Stripe webhook | `UPDATE … WHERE status='pending'` is idempotent. |
| Stolen Drift JWT | Impersonate the user for up to 7 days | Shorten access-token TTL and enforce refresh rotation/revocation. |
| Underage publisher in `adult` room | Bypass age check | `apps.allow_adult` must be true AND `users.id_verified` must be true. There is no way to bypass without an admin approving an ID submission. |
| Malicious upload (RCE via file content) | Upload a polyglot to `/users/me/avatar` | Content-type and extension are restricted; files are served by Bunny CDN as static, never executed by this binary. Add server-side virus scanning if your tenant niche demands it. |

## Reporting a vulnerability

Email the maintainer listed on the GitHub repo, or open a private issue. Do not disclose publicly until a fix is shipped.
