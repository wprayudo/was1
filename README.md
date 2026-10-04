# Genesis Platform

Genesis is a Rust workspace and React frontend for a community banking and asset marketplace platform. The backend contains initial domain types, persistence, application rules, and REST endpoints for the staged areas 1.x–7.b. Treat financial, AML, identity, and marketplace workflows as development scaffolding until provider reconciliation, operational controls, and security review are complete.

## Run locally

Prerequisites: Rust stable and Docker Compose. Node.js 20+ and npm are needed for the frontend. Start SurrealDB with `docker compose up -d`, copy `.env.example` to `.env`, and replace the JWT secret before starting the API. The API applies tracked database migrations on startup. Start the optional local Zenoh router with `docker compose --profile realtime up -d`; the compose mapping binds port 7447 to loopback only. Unset `ZENOH_ENDPOINT` to run without it. ZeroClaw, OpenClaw, and Flow-Like are optional HTTP adapters. QuDAG is not part of the local backend runtime.

```sh
cargo run -p genesis-interfaces
cd frontend && npm install && npm run dev
```

Copy `.env.example` to `.env` and configure secrets before exposing the service. Never use development credentials in production.

## Workspace

- `genesis-core`: initial domain types for users, accounts, assets, currencies, transfers, communities, guarantees, settlement, payments/AML, and marketplace
- `genesis-infrastructure`: persistence and external system adapters
- `genesis-application`: use cases and service boundaries
- `genesis-interfaces`: Axum HTTP API
- `frontend`: React + TypeScript application shell

The transfer use case validates positive amounts, distinct accounts, ownership, account state, currency and available balance, then debits/credits accounts and records the transfer in one SurrealDB transaction. Transfers, marketplace orders, community fund deposits, community currency adjustments and wallet transfers accept `Idempotency-Key`; a retry reuses the recorded operation, while reusing a key with different parameters is rejected.

## Current implementation boundary

Implemented: domain data structures and validation services, atomic account transfer transaction, account and transfer history endpoints, SurrealDB WebSocket connection and tracked migrations, user registration/login with Argon2 and JWT, authenticated asset endpoints, BM25 marketplace search, listing submission with optional ZeroClaw screening and admin moderation, categories, wishlist/cart/orders/reviews/disputes/analytics routes, admin banking configuration CRUD, AML alert routes, rate limits, optional HTTP/HTTPS listener, Zenoh event publisher, and optional ZeroClaw/OpenClaw/Flow-Like HTTP adapters.

Provider checkout, full refunds, and reconciliation use a normalized HTTP adapter. Configure channel URL, API key, webhook callback URL, and return URL in the superadmin panel; legacy provider credentials and callback URLs in `.env` are fallback-only. The provider implements `GET /health`, `POST /checkout-sessions`, `POST /refunds`, `GET /checkout-sessions?id=...`, and `GET /refunds?id=...`. Checkout returns `session_id` and an HTTPS `checkout_url`; refund returns `refund_id`; status reads return `event_id`, `status`, `amount_minor`, and `currency_code`. All mutations carry `Idempotency-Key`. Use `POST /api/v1/payments/intents/{id}/reconcile` to recover a missed callback. Refunds reverse a settled full payment only after provider confirmation, and chargebacks reverse it once from a signed webhook or reconciliation response. Each configured provider must implement this small adapter contract and the webhook signature contract below.

Community profit reports now require an HTTPS evidence URL and SHA-256 digest. A different community admin reviews the linked evidence, verifies its digest and attests approval. Zenoh events for account transfers, asset creation, payments, community fund and wallet changes, marketplace orders and disputes are recorded transactionally in a database outbox and retried after Zenoh returns. Delivery is at least once, so consumers should deduplicate the `event_id` in each envelope. The Rust workspace compiles, and all 22 tracked migrations were applied successfully to SurrealDB 2.3.8. Provider credentials and external endpoint configuration are deployment inputs.

## Local ledger flows

- Community funds: members deposit from an active account with `POST /api/v1/communities/{community}/funds` and an `Idempotency-Key`. Community admins approve a loan, then activate it to credit the borrower's account from the matching currency fund. The borrower submits a profit report; another community admin approves it before repayment collects principal plus the agreed share in one transaction.
- Community currency: admins mint or burn to a member wallet using `POST /api/v1/communities/{community}/currencies/{currency}/{mint|burn}`; members transfer balances through `/wallet-transfers`. These updates change wallet and supply together and use idempotency keys.
- Marketplace: purchase debits buyer funds into escrow. Seller processing/shipping does not release the funds. Buyer completion releases the escrow; an admin dispute resolution can release it to the seller or refund the buyer and restore listing inventory.
- Provider payments: create an intent with a user-owned `account_id`. After AML review, the user starts checkout at `/api/v1/payments/intents/{id}/checkout`; a provider can report `settled`, `failed`, `refunded`, or `chargeback` to `/api/v1/webhooks/payments/{channel_code}`. The callback must send `X-Genesis-Signature`, base64 HMAC-SHA256 over UTF-8 `provider\nevent_id\nreference\nstatus\namount_minor\nCURRENCY` using the channel webhook secret configured in the superadmin panel, or the legacy `PAYMENT_WEBHOOK_SECRET` fallback. Provider event IDs are idempotent. Settlement credits and confirmed refund/chargeback debits are applied once to the selected account.
# Superadmin, merchant payment, and connection checks

Copy `.env.example` to `.env` for local development. Set `FIELD_ENCRYPTION_KEY` to a base64-encoded 32-byte key and `GENESIS_SUPERADMIN_BOOTSTRAP_TOKEN` to a random value of at least 32 characters (for example, generate it with `openssl rand -hex 32`). Keep both outside source control and back up the encryption key: provider secrets in SurrealDB cannot be decrypted if that key is lost. Do not use the example development database credentials in production.

Start SurrealDB and the API, open the Vite frontend, create an account, then use **Aktifkan superadmin** once with its registered email and bootstrap token. Log in again after promotion. The bootstrap token is disabled when it is absent and a unique database control record prevents a second claim. The panel requires both a superadmin JWT and a fresh role check against the user record.

Provider configurations are stored in `payment_provider_config`; API keys and webhook signing secrets are AES-256-GCM encrypted with `FIELD_ENCRYPTION_KEY`. Secret values are write-only in the API. Provider URLs and callback/return URLs must use HTTPS. Configure each merchant to expose the Genesis normalized adapter contract (`GET /health`, `POST /checkout-sessions`, `POST /refunds`, `GET /checkout-sessions?id=…` or `/refunds?id=…`) and sign callbacks using the configured webhook secret.

The superadmin diagnostics verify the running HTTP API, the SurrealDB WebSocket query, and a Zenoh session using `ZENOH_ENDPOINT`. A Zenoh result of `not_configured` means no endpoint was set. The frontend's Vite proxy forwards API and health requests to `127.0.0.1:3000` during local development.
