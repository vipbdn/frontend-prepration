# System Design Notes — Gaming & Rewards Platform + BailBot AI Calling Platform

Nov 26, 2026 · @Vipin

## How to use these notes in an interview

Present each project as "the system I worked on, and how I would design it end to end". You owned the frontend, so speak first-hand about the UI, state and API layer, then walk the backend as the design you integrated with and understand.

**Honest framing line to use:** "I led the frontend, but I worked closely with the backend and the APIs, so let me walk you through the whole system and where my part fits."

### The 6-step framework (use for both projects, about 35-40 minutes)

1. **Clarify requirements (3-5 min)** - functional (what it does) and non-functional (latency, availability, consistency, security).
2. **Estimate scale (2-3 min)** - users, requests per second, storage. Round numbers are fine; show the reasoning.
3. **Define APIs (3-5 min)** - the main REST endpoints, webhooks and real-time events.
4. **High-level design (8-10 min)** - draw the boxes: client, CDN, gateway, services, database, cache, queue, third parties.
5. **Deep dives (10-15 min)** - pick 2-3 hard parts: wallet consistency, real-time voice latency, retries and idempotency.
6. **Trade-offs and scaling (5 min)** - bottlenecks, failure modes, what you would change at 10x.

### Vocabulary to use naturally

- **Idempotency key** - the same request twice gives one result (payments, webhooks).
- **Double-entry ledger** - every money movement is a debit plus a credit; the balance is derived.
- **Eventual vs strong consistency** - leaderboards can lag; wallet balances cannot.
- **Horizontal scaling** - stateless services behind a load balancer.
- **Back-pressure and queues** - absorb spikes (outbound call campaigns, bonus payouts).
- **Webhook** - a third party calls your HTTP endpoint when an event happens (VAPI call ended, payment succeeded).
- **p95 latency** - 95% of requests are faster than this number.

## Project 1: Online Gaming & Rewards Platform

The core idea: users log in, play games, earn coins in a wallet, and redeem bonuses. **The wallet is the hardest part.** It must never lose money or double-credit, so it gets strong consistency, while leaderboards and game catalogs can be cached and eventually consistent.

**Your role (from the resume):** React.js + Redux Toolkit + Bootstrap 5 UI from Figma; Axios integration for auth, games, wallet and bonus redemption APIs.

### 1. Requirements

**Functional**

- Sign up, log in (email/phone OTP, social), session refresh.
- Browse a game catalog by category; launch a game.
- Wallet: view balance, deposit, earn rewards from games, withdraw.
- Bonuses: welcome bonus, daily rewards, promo codes, redeem bonus into wallet.
- Transaction history; leaderboard; notifications.
- Admin panel: manage games, bonus campaigns, users, fraud flags.

**Non-functional**

- **Consistency:** wallet balances strongly consistent, no double spend.
- **Availability:** 99.9%+ for login and game browsing.
- **Latency:** catalog under 200 ms p95 (cached); wallet operations under 500 ms.
- **Security:** JWT auth, rate limits, fraud and abuse checks on bonuses.
- **Auditability:** every coin movement traceable.

### 2. Back-of-envelope estimates (assumed numbers, say so in the interview)

| Metric | Assumption | Result |
| --- | --- | --- |
| Daily active users | 1 million | - |
| Requests per user per day | 100 | 100M requests/day, about 1,200 RPS average, about 5,000 RPS peak |
| Wallet transactions per user per day | 20 | 20M/day, about 230 TPS average, about 1,000 TPS peak |
| Size of one ledger row | about 200 bytes | about 4 GB/day, about 1.5 TB/year |
| Read-to-write ratio | about 10:1 | read-heavy: cache catalog, leaderboard, profile |

### 3. High-level components

&#91;embedded content: Gaming platform architecture · client, gateway, 6 services, data and async layers\]

Read it top down: the React app loads assets from the CDN and calls one gateway; stateless services sit behind it; PostgreSQL holds the ledger, Redis serves fast reads, and the queue fans events out to notifications and fraud checks. Payment webhooks land on the wallet ledger.

| Component | Responsibility | Tech choice |
| --- | --- | --- |
| Web client (React SPA) | UI, routing, state, token handling | React, Redux Toolkit, Axios, Bootstrap 5 |
| CDN | Static JS/CSS, game assets, images | CloudFront / Cloudflare |
| API gateway / load balancer | TLS, routing, rate limiting, JWT check | Nginx / AWS ALB + API Gateway |
| Auth service | Sign-up, login, OTP, JWT + refresh tokens | Node.js, Redis for OTP and sessions |
| User service | Profile, KYC status, preferences | Node.js + PostgreSQL |
| Game catalog service | Game list, categories, launch URLs | Node.js + PostgreSQL, Redis cache |
| Game session service | Start/end session, score, result validation | Server-authoritative results; WebSocket for live games |
| Wallet / ledger service | Balance, credits, debits, idempotency | PostgreSQL (ACID), double-entry ledger |
| Bonus / rewards service | Campaign rules, eligibility, redemption | Rules engine, writes through the wallet service |
| Payment service | Deposits and withdrawals via gateway | Razorpay / Stripe + webhooks |
| Leaderboard service | Top-N ranks | Redis sorted sets (ZADD, ZREVRANGE) |
| Notification service | Email, SMS, push, in-app | Queue consumer + FCM / SES / SMS provider |
| Message queue | Async events: game ended, bonus granted | Kafka / RabbitMQ / SQS |
| Analytics + fraud | Event stream, anomaly rules | Kafka to warehouse; rule checks |
| Admin panel | Campaigns, games, users | React + role-based access |

### 4. Key API design

| Method + path | Purpose | Notes |
| --- | --- | --- |
| POST /auth/login | Log in | Returns access token (15 min) + refresh token (httpOnly cookie) |
| POST /auth/refresh | New access token | Called by the Axios interceptor on 401 |
| GET /games?category=&page= | Game catalog | Cached; cursor or page pagination |
| POST /games/{id}/sessions | Start a game session | Returns sessionId |
| POST /sessions/{id}/result | Submit a result | Server validates; triggers reward event |
| GET /wallet | Balance | Read from the account balance row |
| GET /wallet/transactions?cursor= | History | Cursor pagination, newest first |
| POST /wallet/deposit | Start a deposit | Header Idempotency-Key; returns payment order |
| POST /webhooks/payment | Gateway callback | Verify signature; idempotent on gateway txn id |
| POST /bonuses/{id}/redeem | Redeem a bonus | Idempotency-Key; eligibility check |
| GET /leaderboard?period=weekly | Top players | From Redis |

### 5. Deep dive: wallet and ledger (the question interviewers love)

**Problem:** two requests redeem the same bonus at once, or a payment webhook arrives twice. Without care, the user is credited twice.

**Design:**

1. **Double-entry ledger.** Never just `UPDATE balance = balance + 100`. Write two ledger rows (debit the bonus pool account, credit the user wallet account) in one DB transaction.
2. **Idempotency key.** The client sends a UUID in an `Idempotency-Key` header. The server stores it with a unique constraint; a repeat returns the saved response instead of crediting again.
3. **Row locking or optimistic locking.** `SELECT ... FOR UPDATE` on the wallet row, or a `version` column checked in the update, so concurrent debits cannot overdraw.
4. **Balance check before debit.** A withdrawal fails if balance minus amount is below zero, inside the same transaction.
5. **Outbox pattern.** In the same transaction, write a `wallet_events` outbox row; a worker publishes it to Kafka (notifications, analytics). This avoids "DB saved but event lost".
6. **Reconciliation job.** A nightly job sums the ledger per account and compares it with the cached balance and with gateway settlement reports.

**Data model (simplified)**

| Table | Key columns |
| --- | --- |
| users | id, email, phone, kyc\_status, created\_at |
| wallet\_accounts | id, user\_id, currency (COIN / INR), balance, version |
| ledger\_entries | id, txn\_id, account\_id, direction (DEBIT / CREDIT), amount, created\_at |
| transactions | id, type (DEPOSIT / REWARD / BONUS / WITHDRAW), status, idempotency\_key (unique), reference\_id |
| bonuses | id, type, amount, rules\_json, starts\_at, ends\_at, max\_per\_user |
| user\_bonuses | user\_id, bonus\_id, status, redeemed\_at (unique on user\_id + bonus\_id) |
| games | id, name, category, thumbnail\_url, launch\_url, is\_active |
| game\_sessions | id, user\_id, game\_id, started\_at, ended\_at, score, result |

Store money as integers (paise or coin units), never floats.

### 6. Deep dive: bonus redemption flow

1. User clicks Redeem; the frontend disables the button and generates an Idempotency-Key.
2. Bonus service checks eligibility: campaign active, user not already redeemed (unique constraint on user\_id + bonus\_id), KYC or deposit conditions met.
3. Bonus service calls the wallet service to credit, passing the same idempotency key.
4. Wallet writes the ledger rows and outbox event in one transaction.
5. Response returns the new balance; the frontend updates the Redux wallet slice; a notification goes out asynchronously.

**Fraud controls:** rate limit per user and IP, device fingerprinting, one bonus per device or phone, wagering requirements before withdrawal, velocity rules (many accounts from one IP).

### 7. Deep dive: leaderboard

- Redis sorted set per period: `leaderboard:weekly:2026-W48`. On each result, `ZINCRBY` the user's score.
- Top 100: `ZREVRANGE key 0 99 WITHSCORES`. A user's rank: `ZREVRANK`. Both O(log N).
- Expire old period keys; snapshot final results to PostgreSQL for history and prizes.
- Eventually consistent is fine here: a few seconds of lag is acceptable.

### 8. Frontend architecture (your strongest area, go deep here)

- **Structure:** feature folders (auth, games, wallet, bonus), shared reusable components built from Figma (Button, Card, Modal, GameTile).
- **State:** Redux Toolkit slices for `auth`, `wallet`, `games`, `bonus`; `createAsyncThunk` for API calls with `pending / fulfilled / rejected` states for loaders and errors.
- **Server data vs UI state:** catalog and history are server data (could also use RTK Query or TanStack Query for caching and refetch); modals and filters are local UI state.
- **Axios instance + interceptors:** request interceptor adds the Bearer token; response interceptor catches 401, calls `/auth/refresh` once, queues other failed requests, then retries them; global error toast.
- **Wallet UI correctness:** never compute balance on the client; always show the server's balance after a mutation. Disable buttons during requests to avoid double submit; the idempotency key protects the server anyway.
- **Performance:** route-level code splitting with `React.lazy`, lazy-loaded game thumbnails, CDN images, memoised lists (`React.memo`, `useMemo`), virtualised long lists for history.
- **Security:** access token in memory, refresh token in an httpOnly cookie (protects against XSS theft); never trust client-side game scores.
- **Real-time:** WebSocket or Server-Sent Events for live balance changes, leaderboard updates and notifications.

### 9. Scaling and reliability

- Stateless services behind a load balancer; scale horizontally with auto-scaling.
- PostgreSQL primary for writes plus read replicas for history and reports; partition `ledger_entries` by month; shard by user\_id at very large scale.
- Redis for cache (catalog, profile) with TTL and cache invalidation on admin updates.
- Queue absorbs spikes (tournament end, campaign launch) and decouples notifications.
- Circuit breakers and timeouts around the payment gateway; retries with exponential backoff.
- Monitoring: Prometheus/Grafana or Datadog; alerts on wallet errors, ledger mismatch, payment failure rate.

### 10. Trade-offs to mention

| Decision | Chosen | Why / trade-off |
| --- | --- | --- |
| Wallet DB | SQL (PostgreSQL) | ACID transactions and constraints beat NoSQL flexibility for money |
| Leaderboard | Redis sorted set | Fast ranks; can lose recent writes on crash, so snapshot to DB |
| Sync vs async | Wallet credit sync; notifications async | User needs the balance now; emails can wait |
| Monolith vs microservices | Start as a modular monolith, split wallet and game services later | Fewer moving parts early; split when teams or load grow |
| Game results | Server-authoritative | Prevents cheating; costs more server work |

## Project 2: BailBot - AI Voice Calling & CRM Platform for Bail Bond Agencies

The core idea: when someone is arrested, a family member calls a bail bond agency, often at 2 a.m. BailBot's AI voice agent answers 24/7, collects the defendant and caller details, answers common questions, saves the lead to the CRM, and transfers to a live bail agent when needed. **The hardest parts are real-time voice latency and reliable hand-off of data to the CRM.**

**Your role (from the resume):** led the frontend (React.js, MUI, Redux Toolkit, Formik/Yup, Axios interceptors); built AI voice assistants on VAPI that capture caller details and save leads to GoHighLevel CRM.

> Note: bailbot.ai renders its content with JavaScript, so its feature list could not be read for these notes. The design below is based on your resume and a [public Chetu case study](https://www.chetu.com/white-paper/voice-ai-agent-for-bail-bond-hyperautomation-solution.php?t=1) of a Voice AI agent for a 24/7 Colorado bail bond company (GoHighLevel CRM, Simply Bail, Python/Django REST, PostgreSQL, agent dashboard, eSignature, payment gateway). Check the details against what you actually built before the interview.

### 1. Requirements

**Functional**

- **Inbound AI calls 24/7:** greet, ask who was arrested, which jail/county, booking number, charges, bond amount, caller name, relationship, phone, best time to call.
- **Answer FAQs:** how bail works, fees (typically a percentage of the bond), payment plans, documents needed, office hours.
- **Save leads** to GoHighLevel CRM as a contact plus an opportunity in a pipeline stage (New Lead, Qualified, Contract Sent, Bonded).
- **Warm transfer** to an on-call bail agent when the caller asks for a human, the bond is large, or the AI cannot help.
- **Outbound calls and SMS:** follow-up on missed leads, court-date reminders to defendants, payment-plan reminders, check-in calls.
- **Agent dashboard:** list of calls, transcripts, recordings, extracted lead fields, call outcome, assistant configuration (prompt, voice, business hours, transfer numbers).
- **Multi-tenant:** each agency has its own phone number, assistant, CRM location and users.

**Non-functional**

- **Low latency:** the AI must reply within about 1 second of the caller finishing, or the call feels robotic.
- **High availability:** a missed 2 a.m. call is a lost customer; 99.9%+.
- **Reliability of data:** no lead lost even if the CRM API is down (retry queue).
- **Concurrency:** many simultaneous calls (for example, after weekend arrests).
- **Security and compliance:** PII (names, charges, phone numbers) encrypted; call-recording disclosure; consent for outbound calls and SMS (TCPA in the US); audit logs.

### 2. Back-of-envelope estimates (assumed)

| Metric | Assumption | Result |
| --- | --- | --- |
| Agencies (tenants) | 200 | - |
| Calls per agency per day | 50 inbound + 100 outbound reminders | 30,000 calls/day |
| Average call length | 3 minutes | 90,000 call-minutes/day; minutes drive cost |
| Peak concurrent calls | about 5% of daily calls in the busiest hour, spread over 60 min x 3 min calls | about 75 concurrent calls; design for 200+ |
| Webhook events per call | about 10 (status, tool calls, transcript, end report) | about 300,000 webhook events/day, low RPS but bursty |
| Recording size | about 1 MB/min (compressed) | about 90 GB/day in object storage, lifecycle to cold storage |

### 3. What VAPI does vs what you build

VAPI is a voice-AI orchestration platform. It glues together telephony, speech-to-text, an LLM and text-to-speech, and handles real-time things like turn-taking and interruptions. You configure an **assistant** (system prompt, voice, model, tools) and point its **server URL** at your backend.

| Layer | Handled by | Examples |
| --- | --- | --- |
| Phone number / PSTN | Telephony provider via VAPI | Twilio / Vonage / VAPI number |
| Speech-to-text (STT) | VAPI + provider | Deepgram |
| LLM reasoning | VAPI + provider | OpenAI GPT-4o class model |
| Text-to-speech (TTS) | VAPI + provider | ElevenLabs, PlayHT |
| Turn-taking, interruptions, endpointing | VAPI | Detects when the caller stops talking; barge-in |
| Business logic (tools / functions) | **Your backend** via webhooks | saveLead, checkBusinessHours, transferToAgent, lookupInmate |
| CRM sync | **Your backend** | GoHighLevel contacts, opportunities, notes, tags |
| Dashboard, auth, configuration | **Your frontend + backend** | React + MUI admin app |
| Scheduling outbound calls | **Your backend** | Scheduler + queue calling the VAPI create-call API |

### 4. High-level components

| Component | Responsibility | Tech choice |
| --- | --- | --- |
| Caller / defendant phone | Inbound and outbound PSTN calls | - |
| Telephony + VAPI | Media streaming, STT, LLM, TTS, tool-call events | VAPI, Twilio, Deepgram, OpenAI, ElevenLabs |
| API gateway | TLS, auth, rate limits, webhook signature check | Nginx / AWS API Gateway |
| Webhook / tool service | Receives VAPI server messages; runs tools fast; returns results | Node.js or Django REST; stateless, auto-scaled |
| Call service | Stores call records, transcripts, summaries, outcomes | PostgreSQL |
| Lead service | Validates and normalizes extracted fields; dedupes by phone | PostgreSQL + Redis |
| CRM sync worker | Pushes contacts, opportunities, notes to GoHighLevel with retries | Queue consumer; GHL REST API with OAuth per location |
| Scheduler | Court-date and payment reminders, missed-call follow-ups | Cron / BullMQ / Celery beat |
| Outbound dialer worker | Pulls jobs, respects concurrency, business hours and consent, calls VAPI API | Queue consumer |
| Message queue | Decouples webhooks from slow work (CRM, email, analytics) | SQS / RabbitMQ / Redis (BullMQ) |
| Object storage | Call recordings, documents, contracts | S3 with encryption + signed URLs |
| Notification service | SMS/email alert to on-call agent on a new hot lead | Twilio SMS, email, GHL workflows |
| Agent dashboard (frontend) | Calls, transcripts, leads, assistant settings, analytics | React.js, MUI, Redux Toolkit, Formik/Yup, Axios |
| Integrations | Bail management software, eSignature, payment gateway | Simply Bail, DocuSign-style tools, Stripe |
| Observability | Logs, metrics, call-quality alerts | Datadog / Grafana, Sentry |

### 5. Inbound call flow (walk this step by step in the interview)

1. Caller dials the agency's number. Telephony routes it to VAPI, which loads that number's assistant.
2. Optionally VAPI sends an `assistant-request` webhook; your backend returns a tenant-specific assistant (agency name, hours, transfer number, prompt).
3. AI greets: "Thank you for calling ABC Bail Bonds, this call may be recorded. Who are you calling about?"
4. Each turn: caller speech goes to STT (streaming text), then to the LLM with the system prompt and conversation history, then the reply goes to TTS and is streamed back.
5. When the LLM has the details, it calls a **tool**, for example `saveLead({defendantName, jail, bookingNumber, bondAmount, callerName, phone, relationship})`. VAPI sends a `tool-calls` webhook to your server.
6. Your webhook service validates the fields, writes to PostgreSQL, puts a `lead.created` job on the queue, and **returns within a few hundred milliseconds** so the AI can say "Got it, I've saved your details."
7. If the caller wants a human or rules say so, the AI calls `transferCall` to the on-call agent's number (warm transfer with a short summary).
8. Call ends. VAPI sends an `end-of-call-report` with transcript, recording URL, summary, duration and cost. Your backend stores it and updates the lead.
9. The CRM sync worker creates/updates the GoHighLevel contact, creates an opportunity in the pipeline, adds the summary as a note, and tags the lead. A GHL workflow can then send an SMS and alert the agent.
10. The dashboard shows the new call and lead (polling, WebSocket or Server-Sent Events).

### 6. Outbound reminders flow

1. Agent sets a court date or payment plan in the dashboard (or it syncs from Simply Bail / the CRM).
2. Scheduler creates reminder jobs: for example 7 days, 1 day and morning-of the court date.
3. Dialer worker picks due jobs and checks: consent on file, allowed calling hours in the defendant's time zone, do-not-call list, tenant concurrency limit.
4. It calls the VAPI create-call API with an outbound assistant and variables (name, court, date, time, address).
5. Outcome comes back by webhook: answered and confirmed, voicemail left, no answer, wrong number. Retry with backoff; fall back to SMS.
6. Missed court-date confirmations are escalated to the agent (risk of bond forfeiture).

### 7. Deep dive: latency budget

A natural conversation needs about 800 ms to 1.2 s from the caller stopping to the AI starting to speak. Rough budget:

| Stage | Typical target |
| --- | --- |
| Endpointing (detect end of speech) | 200-400 ms |
| STT final transcript (streaming) | 100-300 ms |
| LLM first token | 300-600 ms |
| TTS first audio chunk | 100-250 ms |
| Your tool webhook (when called) | under 300-500 ms |

**How to keep it low:** streaming at every stage; short system prompts; a fast model; keep tool handlers fast (write to DB and queue, never call the slow CRM synchronously); host the webhook server in the same region as VAPI; keep warm instances (no cold starts); filler phrases ("Let me note that down...") for slow tools.

### 8. Deep dive: reliability and idempotency

- **Webhooks can be retried or arrive twice.** Store the VAPI call ID + message type + tool-call ID with a unique constraint; ignore duplicates.
- **CRM down or rate-limited.** Never lose the lead: save locally first, then the queue worker retries with exponential backoff; after N failures, move to a dead-letter queue and alert.
- **Dedupe leads by phone number** (normalized to E.164) and update the existing GHL contact instead of creating duplicates.
- **Verify webhook authenticity** with a shared secret or signature header.
- **Fallbacks:** if the AI pipeline fails, forward the call to the agency's human number or voicemail; if transfer fails, take a message and send an urgent SMS to the agent.
- **Timeouts** on every external call; circuit breaker around GHL.

### 9. Prompt and tool design (LLM part)

- **System prompt:** role ("You are the after-hours assistant for ABC Bail Bonds"), goals (collect fields in order), tone (calm, empathetic, short sentences), rules (never give legal advice, never promise release times, quote only configured fees), and when to transfer.
- **Structured extraction:** tools with a JSON schema make the LLM return clean fields instead of free text.
- **Guardrails:** confirm phone numbers by reading them back; spell names; limit topics; hand over to a human on uncertainty.
- **Per-tenant variables:** agency name, hours, fee rules and transfer numbers come from the DB, not hard-coded in the prompt.
- **Evaluation:** review transcripts, track field-completion rate and transfer rate; A/B test prompt versions.

### 10. Data model (simplified)

| Table | Key columns |
| --- | --- |
| tenants (agencies) | id, name, timezone, ghl\_location\_id, ghl\_oauth\_tokens (encrypted), plan |
| phone\_numbers | id, tenant\_id, e164, vapi\_phone\_id, assistant\_id |
| assistants | id, tenant\_id, vapi\_assistant\_id, prompt\_version, voice, transfer\_numbers, business\_hours |
| calls | id, tenant\_id, vapi\_call\_id (unique), direction, from, to, status, started\_at, duration\_sec, cost, recording\_url, summary, outcome |
| transcripts | call\_id, role, text, timestamp (or one JSON blob per call) |
| leads | id, tenant\_id, call\_id, defendant\_name, jail, booking\_no, charges, bond\_amount, caller\_name, relationship, phone, status, ghl\_contact\_id |
| reminders | id, tenant\_id, lead\_id, type (COURT / PAYMENT), due\_at, channel, status, attempts |
| consents | phone, tenant\_id, channel, source, granted\_at, revoked\_at |
| webhook\_events | id, vapi\_call\_id, type, dedupe\_key (unique), payload, processed\_at |
| users | id, tenant\_id, email, role (ADMIN / AGENT) |

### 11. Frontend architecture (your part, go deep)

- **App:** React.js SPA with MUI; layout with sidebar (Dashboard, Calls, Leads, Assistants, Reminders, Settings).
- **State:** Redux Toolkit slices (`auth`, `calls`, `leads`, `assistants`); async thunks for API calls; selectors for filtered views.
- **Forms:** Formik + Yup for assistant settings (prompt, voice, business hours, transfer numbers with phone validation) and lead editing.
- **Axios interceptor setup:** attach JWT and tenant ID to every request; on 401 refresh the token or log out; global error handler shows a snackbar; consistent error shape.
- **Calls page:** server-side pagination, filters (date, outcome, direction), transcript viewer in chat-bubble style, audio player with a signed recording URL.
- **Live updates:** poll every few seconds or use WebSocket/SSE to show in-progress calls and new leads.
- **Reusable components:** DataTable, StatusChip, TranscriptView, PhoneInput, ConfirmDialog.
- **Role-based UI:** admins edit assistants and billing; agents see leads and calls only.
- **Performance:** lazy-load heavy pages (analytics, transcript), memoize table rows, debounce search.

### 12. Security and compliance

- Encrypt PII at rest (DB encryption, S3 SSE) and in transit (TLS).
- Recording disclosure at the start of the call; some US states require all-party consent.
- Outbound calls/SMS only with consent; honor opt-out (STOP) and do-not-call lists; call only in allowed hours.
- Tenant isolation: every query filtered by tenant\_id; row-level security if on PostgreSQL.
- Signed, expiring URLs for recordings; role-based access; audit log of who viewed what.
- Retention policy: delete or archive recordings and transcripts after a set period.
- The AI must not give legal advice; clear hand-off to a licensed agent.

### 13. Scaling

- Webhook service is stateless; auto-scale on CPU and request count; keep it lightweight.
- Move all slow work (CRM, email, analytics, summaries) to queues.
- Outbound dialer uses a token bucket per tenant and global concurrency limits matching the VAPI plan and telephony limits.
- Partition `calls` and `webhook_events` by month; archive old transcripts to cheaper storage.
- Cache tenant configuration (assistant settings, hours) in Redis for fast tool responses.
- Cost control: call-minute usage per tenant, cheaper models for simple flows, cap call length.

### 14. Trade-offs to mention

| Decision | Chosen | Why / trade-off |
| --- | --- | --- |
| Build own voice pipeline vs VAPI | VAPI | Faster to market, handles turn-taking; less control, per-minute cost, vendor lock-in |
| CRM write sync vs async | Async via queue | Keeps tool responses fast and survives CRM outages; CRM is a few seconds behind |
| GoHighLevel as CRM vs own DB only | Both: own DB is source of truth, GHL for agency workflows | Agencies already use GHL; you never lose data if GHL fails |
| Polling vs WebSocket on dashboard | Start with polling, move to WebSocket/SSE | Simpler first; real-time later |
| One prompt vs per-tenant prompts | Template + per-tenant variables | Consistent quality; agencies still customize |

## Likely interview questions with model answers

### Gaming & Rewards platform

**Q: How do you prevent a user from redeeming the same bonus twice?** Three layers. The frontend disables the button and sends an Idempotency-Key. The server stores that key with a unique constraint and returns the saved response on a repeat. The `user_bonuses` table has a unique constraint on (user\_id, bonus\_id), so the database itself rejects a second redemption.

**Q: Why SQL for the wallet and not MongoDB?** Money needs ACID transactions, constraints and row locks. A debit and credit must both succeed or both fail. PostgreSQL gives that natively; I would use NoSQL or Redis for catalog caching and leaderboards instead.

**Q: What happens if the payment gateway webhook arrives twice, or never?** Twice: dedupe on the gateway transaction ID. Never: a reconciliation job polls the gateway for pending orders older than a few minutes and settles them.

**Q: How would you scale the leaderboard to 10 million users?** Redis sorted sets give O(log N) updates and rank lookups. Shard by game or region if one key gets too large; snapshot to the database at period end.

**Q: How does your frontend handle an expired token mid-session?** The Axios response interceptor catches the 401, calls the refresh endpoint once, holds other failed requests in a queue, then replays them with the new token. If refresh fails, it clears state and redirects to login.

### BailBot AI calling platform

**Q: Walk me through what happens when someone calls.** Use the 10-step inbound flow in Project 2, section 5: phone to VAPI, STT, LLM, TTS loop, tool call to our webhook, save locally, queue CRM sync, transfer if needed, end-of-call report, dashboard update.

**Q: Why not call GoHighLevel directly inside the tool call?** The AI is waiting on the line. A slow or failed CRM call creates dead air or a lost lead. We save to our DB in milliseconds, reply to VAPI, and sync GHL asynchronously with retries.

**Q: How do you keep the AI responsive?** Streaming STT, LLM and TTS; a fast model; a short prompt; fast tool handlers in the same region; warm servers; filler phrases for slower tools.

**Q: What if 100 calls come in at once?** VAPI handles the media side up to the plan's concurrency. Our webhook service is stateless and auto-scales; slow work is queued. If we hit a concurrency limit, overflow calls forward to the agency's human line or voicemail.

**Q: How do you handle sensitive data?** Encryption at rest and in transit, tenant isolation on every query, signed URLs for recordings, role-based access, audit logs, retention rules, recording disclosure and consent for outbound calls.

**Q: How would you measure whether the AI agent is good?** Lead-capture rate, required-field completion rate, transfer rate, average call length, caller hang-up rate in the first 10 seconds, and manual transcript review scores.

### General questions that apply to both

**Q: Monolith or microservices?** Start with a modular monolith with clear module boundaries. Split out the piece with different scaling or reliability needs first (wallet; webhook service) when the team or load grows.

**Q: How do you make sure an event is not lost between the database and the queue?** The outbox pattern: write the event to an outbox table in the same transaction, then a worker publishes it and marks it sent.

## One-page cheat sheet

| Topic | Gaming & Rewards | BailBot AI calling |
| --- | --- | --- |
| Hardest problem | Wallet consistency, no double credit | Real-time voice latency + no lost leads |
| Source of truth | PostgreSQL ledger | Own PostgreSQL; GHL is a synced copy |
| Cache | Redis: catalog, profile, leaderboard | Redis: tenant config, dedupe keys |
| Async work | Notifications, analytics, fraud | CRM sync, SMS/email, summaries, outbound dialing |
| Key pattern | Double-entry ledger, idempotency key, outbox | Webhooks, idempotent handlers, retry + dead-letter queue |
| Third parties | Payment gateway, SMS/email | VAPI, Twilio, Deepgram, OpenAI, ElevenLabs, GoHighLevel |
| Real-time to UI | WebSocket for balance and leaderboard | Polling then WebSocket/SSE for live calls and leads |
| Security | JWT + refresh cookie, rate limits, fraud rules | PII encryption, tenant isolation, consent, recording disclosure |
| Your frontend story | Redux slices, Axios refresh interceptor, code splitting | MUI dashboard, Formik/Yup config forms, Axios auth/error interceptor |

**Closing line for either project:** "If I had to take this to 10x, my first bottleneck would be \_\_\_, and I would solve it with \_\_\_." For gaming: the wallet database, by partitioning the ledger and read replicas. For BailBot: concurrent call capacity and CRM rate limits, by queue-based dialing and per-tenant throttling.
