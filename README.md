### THIS IS PROTOTYPE ONLY ###
<div align="center">

# 🏥 MediConsult — Enterprise Telemedicine Platform

**Production-grade medical consultation infrastructure built for scale, security, and compliance**

[![Next.js](https://img.shields.io/badge/Next.js-15.5-black?logo=next.js)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.7-blue?logo=typescript)](https://www.typescriptlang.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-7-DC382D?logo=redis)](https://redis.io/)
[![Prisma](https://img.shields.io/badge/Prisma-6-2D3748?logo=prisma)](https://www.prisma.io/)
[![BullMQ](https://img.shields.io/badge/BullMQ-Queues-red)](https://bullmq.io/)
[![Toxiproxy](https://img.shields.io/badge/Chaos_Engineering-Toxiproxy-orange)](https://github.com/Shopify/toxiproxy)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

> A full-stack telemedicine backend solving hard distributed systems problems: race-condition-free scheduling, stateless real-time pub/sub, transactional outbox delivery, HIPAA-aligned audit enforcement, and chaos-engineering-proven resilience — built to hospital-grade reliability standards.

</div>

---

## 👔 Executive Summary

**What is this?**
MediConsult is a production-ready telemedicine engine — the kind that powers platforms like Teladoc or Practo. It manages the complete patient journey: booking → payment → live consultation with a verified doctor → digital prescription.

**Why is this genuinely hard to build?**

> Imagine a hospital system that processes payments, assigns doctors, and runs live consultations simultaneously across thousands of users. Now imagine what happens when two doctors click "Accept Patient" at the exact same millisecond. Or when the server crashes exactly while a prescription is being sent. Or when the payment gateway sends the same webhook twice.

Every one of these scenarios causes catastrophic data corruption in naïve systems. MediConsult was engineered from the ground up to make all of them *structurally impossible*:

- **Double-booking** is eliminated at the database level using atomic Compare-And-Swap — no application locks needed
- **Lost messages** are impossible using the Transactional Outbox Pattern — events are committed to PostgreSQL atomically with the action that triggered them
- **Duplicate payments** are blocked via database-level idempotency keys on every financial operation
- **System crashes** are survived gracefully — workers recover jobs, outbox events replay, and WebSocket clients reconnect without losing state

The entire system was put through **Chaos Engineering** — using Toxiproxy to deliberately sever database connections, crash worker processes, and corrupt network traffic mid-operation — and proved it recovers completely every time.

---

## 📑 Table of Contents

1. [Executive Summary](#-executive-summary)
2. [Platform Overview](#-platform-overview)
3. [System Architecture](#-system-architecture)
4. [User Roles & Access Control](#-user-roles--access-control)
5. [Consultation Lifecycle & State Machine](#-consultation-lifecycle--state-machine)
6. [End-to-End Workflow](#-end-to-end-workflow)
7. [Real-Time Gateway & Pub/Sub](#-real-time-gateway--pubsub)
8. [Background Worker Architecture](#-background-worker-architecture)
9. [Security Model](#-security-model)
10. [Observability & Metrics](#-observability--metrics)
11. [Database Schema](#-database-schema)
12. [API Reference](#-api-reference)
13. [Tech Stack](#-tech-stack)
14. [Project Structure](#-project-structure)
15. [Key Engineering Decisions](#-key-engineering-decisions)
16. [System Invariants](#-system-invariants)
17. [Testing & Chaos Engineering](#-testing--chaos-engineering)
18. [Quick Start](#-quick-start)
19. [Deployment](#-deployment)

---

## 🌟 Platform Overview

| Capability | Implementation |
|---|---|
| 🔐 **Zero-Trust Security** | HTTP-only sessions, CSRF double-submit cookie+header, per-route RBAC guards |
| ⚛️ **Atomic State Machine** | Consultation transitions via SQL CAS with version-stamped rows — no distributed locks |
| 🧾 **Immutable Audit Logs** | Prisma ORM extension blocks `update`/`delete` on `AuditLog` at the library layer |
| 💳 **Idempotent Payments** | Razorpay integration with `externalEventId UNIQUE` dedup on all webhook events |
| 🌐 **Stateless Real-Time Gateway** | Separate WebSocket server backed by Redis Pub/Sub — horizontally scalable |
| ✉️ **Transactional Outbox** | Side effects committed in same DB transaction as the triggering state change |
| 📋 **Hybrid File Storage** | `StorageProvider` interface auto-selects S3 or local disk based on env credentials |
| ⚡ **Redis Backpressure** | Per-endpoint concurrency caps via `INCR`/`DECR` with `finally` block guarantee |
| 📨 **8 Async BullMQ Workers** | Assignment, notification, refund, refund-recovery, webhook, outbox-recovery, sweeper |
| 📊 **Prometheus Observability** | Counters, histograms, gauges for every layer — HTTP, socket, Redis, queue, outbox |
| 🎭 **Chaos-Proven Resilience** | 5 test profiles, 30+ scenarios, Toxiproxy fault injection, Docker SIGKILL recovery |

---

## 🏗 System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        P["👤 Patient Browser"]
        D["🩺 Doctor Browser"]
        A["⚙️ Admin Browser"]
    end

    subgraph "Edge Layer"
        MW["Next.js Middleware\n━━━━━━━━━━━━━━━\nCSRF double-submit\nx-method header injection\nCorrelation ID propagation"]
    end

    subgraph "Application Layer — Next.js 15 App Router  :3000"
        API["18 REST API Route Groups"]
        RBAC["RBAC Guards\nrequireAuth → requireRole\n→ requireVerifiedDoctor\n→ canAccessConsultation"]
        SM["State Machine (Pure Function)\ntransition(session, event)\n→ ConsultationEvents\n→ OperationalEvents\n→ OutboxEvents"]
        OBS["Observability\nwithTrace() / withObservability()\nPino logs · Prometheus metrics\nAsyncLocalStorage traceId"]
    end

    subgraph "Real-Time Layer — Standalone Process  :3001"
        GW["WebSocket Gateway\nHTTP Upgrade + Auth Ticket\nGateway State: CONNECTED ↔ DEGRADED"]
        CM["ConnectionManager\nuserId → socketId → Connection\n30s heartbeat · 512KB backpressure"]
        SUB["Redis Subscriber\nChannel: realtime:events:user:[id]\nZod schema-validated fanout"]
    end

    subgraph "Data Layer"
        PG[("🐘 PostgreSQL 16\nPrimary Source of Truth\n28 models · ACID Transactions")]
        REDIS[("🔴 Redis 7\nSessions · Backpressure\nBullMQ · Pub/Sub · Locks")]
    end

    subgraph "Async Worker Layer — Standalone Process"
        RELAY["OutboxRelay\n2s poll · batch=50\nSKIP LOCKED two-phase claim"]
        WORKERS["8 BullMQ Workers\nassignment · notification · refund\nrefund-recovery · webhook\nwebhook-recovery · outbox-recovery · sweeper"]
        SWEEPERS["Anomaly Sweepers\npaid-unassigned · orphan-payment\nassignment-lease · outbox-stuck\npayment-timeout"]
    end

    subgraph "External"
        RAZORPAY["💳 Razorpay\nPayment + Refund + Webhooks"]
        S3["☁️ AWS S3 / Local Disk\nMedical File Storage"]
    end

    P & D & A --> MW --> API --> RBAC --> SM --> OBS
    SM --> PG
    API --> REDIS
    GW --> CM --> SUB --> REDIS
    RELAY --> PG
    RELAY --> REDIS
    WORKERS --> PG & REDIS
    SWEEPERS --> PG & REDIS
    API --> RAZORPAY & S3
```

---

## 👤 User Roles & Access Control

```mermaid
graph LR
    subgraph "PATIENT"
        P1["📋 Book consultation"]
        P2["💳 Pay via Razorpay"]
        P3["📁 Upload medical reports"]
        P4["💬 Chat during ACTIVE sessions"]
        P5["📄 View prescriptions & test results"]
        P6["👁️ Own consultations only"]
    end

    subgraph "DOCTOR — Verified + AVAILABLE"
        D1["📋 View priority-sorted patient queue"]
        D2["⚡ Accept patient via atomic CAS"]
        D3["▶️ Start & close consultation sessions"]
        D4["💬 Chat · 📋 View medical reports"]
        D5["💊 Issue versioned prescriptions"]
        D6["🔬 Order diagnostic tests"]
        D7["🕐 Manage availability & shift end"]
    end

    subgraph "ADMIN"
        A1["🎯 Manual doctor assignment"]
        A2["✅ Verify / Reject doctor profiles"]
        A3["📊 Live KPI dashboard (30s refresh)"]
        A4["📥 Financial CSV export"]
        A5["🔍 Audit log viewer"]
        A6["🚫 Zero access to medical content"]
    end
```

**Guard Composition Chain:**
```
requireAuth() → validates HTTP-only session (Redis cache → DB fallback)
  └─ requireRole('DOCTOR') → exact role match
       └─ requireVerifiedDoctor() → DoctorProfile.verificationStatus = 'VERIFIED'
  └─ canAccessConsultation(id) → must be the patient OR the assigned doctor
```

---

## 🔄 Consultation Lifecycle & State Machine

The consultation lifecycle is governed by a **strict, server-enforced, pure-function state machine**. Every transition is atomic at the PostgreSQL level using Compare-And-Swap. Invalid transitions throw at the pure function before ever touching the database.

```mermaid
stateDiagram-v2
    direction LR

    [*] --> PAYMENT_PENDING : Patient submits request

    PAYMENT_PENDING --> PAID : Razorpay webhook captured
    PAYMENT_PENDING --> EXPIRED : Payment timeout (BullMQ)
    PAYMENT_PENDING --> CANCELLED : Patient cancels

    PAID --> READY : Assignment worker assigns doctor
    PAID --> EXPIRED : Worker timeout
    PAID --> CANCELLED : Admin/Worker cancels
    PAID --> REVIEW_PENDING : Assignment exhausted — escalated

    READY --> ACTIVE : Doctor starts session

    ACTIVE --> COMPLETED : Doctor closes consultation
    ACTIVE --> EXPIRED : Session timeout

    COMPLETED --> [*] : Immutable snapshot created
    CANCELLED --> [*]
    EXPIRED --> [*]
```

**Cancellation Authority Matrix:**
| Actor | Can Cancel |
|---|---|
| `PATIENT` | `PAYMENT_PENDING` only |
| `WORKER / SYSTEM` | `PAYMENT_PENDING`, `PAID` |
| `ADMIN` | `PAYMENT_PENDING`, `PAID`, `READY` |

**Auto-Refund Trigger:** When `PAID → EXPIRED` or `PAID → CANCELLED`, the pure state machine function automatically emits a `REFUND_REQUESTED` outbox event *within the same DB transaction* — ensuring no patient is ever charged without service.

---

## ⏱ End-to-End Workflow

```mermaid
sequenceDiagram
    participant Patient
    participant NextAPI as Next.js API :3000
    participant PG as PostgreSQL
    participant Relay as OutboxRelay
    participant BullMQ
    participant Assignment as AssignmentWorker
    participant Doctor
    participant Razorpay
    participant Gateway as WS Gateway :3001
    participant Redis

    Patient->>NextAPI: POST /api/consultations
    NextAPI->>PG: INSERT ConsultationSession (PAYMENT_PENDING) + OutboxEvent
    NextAPI-->>Patient: {consultationId, paymentOrderId}

    Patient->>Razorpay: Complete payment
    Razorpay->>NextAPI: POST /api/payments/webhook (HMAC verified)
    NextAPI->>PG: UPDATE status→PAID + INSERT OutboxEvent(ASSIGNMENT_REQUESTED)

    Relay->>PG: Claim OutboxEvent (SKIP LOCKED batch)
    Relay->>BullMQ: Enqueue assignment job
    Relay->>PG: Mark event DISPATCHED

    BullMQ->>Assignment: Execute assignment job
    Assignment->>PG: Score doctors by capacity + recency
    Assignment->>PG: UPDATE ConsultationSession WHERE status='PAID' (CAS — race-safe)
    Assignment->>Redis: PUBLISH realtime:events:user:[patientId]
    Assignment->>Redis: PUBLISH realtime:events:user:[doctorId]

    Gateway->>Patient: Push CONSULTATION_STATUS_CHANGED event
    Gateway->>Doctor: Push QUEUE_UPDATED event

    Doctor->>NextAPI: POST /api/consultations/[id]/start
    NextAPI->>PG: UPDATE status→ACTIVE WHERE status='READY' (CAS)

    Doctor->>NextAPI: POST /api/chat message
    NextAPI->>PG: INSERT ChatMessage + OutboxEvent(REALTIME_DISPATCH)
    Relay->>Redis: PUBLISH CHAT_MESSAGE_CREATED to both users
    Gateway->>Patient: Push real-time chat message

    Doctor->>NextAPI: POST /api/consultations/[id]/complete
    NextAPI->>PG: UPDATE status→COMPLETED + INSERT ConsultationSnapshot
```

---

## 🌐 Real-Time Gateway & Pub/Sub

The platform runs a **fully stateless WebSocket server** as a separate Node.js process on port 3001, completely independent of the Next.js API layer. This enables independent horizontal scaling.

```mermaid
flowchart LR
    Client["Browser Client\n(Patient / Doctor)"]
    
    subgraph "Gateway Process :3001"
        Upgrade["HTTP Upgrade\n+ Auth Ticket Validation"]
        CM["ConnectionManager\nuserId → socketId → Socket\nHeartbeat · Backpressure"]
        Sub["Redis Subscriber\nZod-validated event fanout"]
    end

    subgraph "Event Sources"
        API["Next.js API\n(HTTP action)"]
        Relay["OutboxRelay\n(async dispatch)"]
    end

    Redis[("Redis Pub/Sub\nrealtime:events:user:[id]")]

    Client -- "1 WS Connect + ticket" --> Upgrade
    Upgrade -- "2 Add to registry" --> CM
    CM -- "3 Subscribe" --> Sub
    Sub -- "4 Listen" --> Redis
    API -- "5 PUBLISH" --> Redis
    Relay -- "5 PUBLISH" --> Redis
    Sub -- "6 Fanout to sockets" --> CM
    CM -- "7 Push frame" --> Client
```

**Connection Protocol:**
1. Client calls `GET /api/auth/realtime-ticket` — receives a one-time UUID stored in Redis (TTL: 30s)
2. Client opens `ws://gateway:3001?ticket=<uuid>` — gateway redeems ticket from Redis (single-use)
3. Gateway sends `CONNECTION_ESTABLISHED` control frame. Clients **must not subscribe** until this is received — eliminates reconnect race conditions
4. Gateway broadcasts `SYSTEM_DEGRADED` if Redis goes down, transitions to `DEGRADED` state, and recovers automatically

**Real-Time Event Types** (all Zod schema-validated):

| Event | Trigger |
|---|---|
| `CONSULTATION_STATUS_CHANGED` | Any state machine transition (carries `consultationVersion`) |
| `CHAT_MESSAGE_CREATED` | New chat message persisted to DB |
| `QUEUE_UPDATED` | Queue length changes |
| `PRESCRIPTION_UPDATED` | Doctor issues or revises prescription |
| `SESSION_REVOKED` | Session invalidated server-side |

---

## ⚙️ Background Worker Architecture

```mermaid
flowchart TD
    DB[("PostgreSQL\nOutboxEvent table")]
    Relay["OutboxRelay\nSELECT ... FOR UPDATE SKIP LOCKED\nbatch=50, poll=2s"]
    
    DB -->|"Claim PENDING events"| Relay
    
    Relay -->|"ASSIGNMENT_REQUESTED"| AQ["Assignment Queue"]
    Relay -->|"NOTIFICATION_REQUIRED"| NQ["Notification Queue"]
    Relay -->|"REFUND_REQUESTED"| RQ["Refund Queue"]
    Relay -->|"REALTIME_DISPATCH"| Redis["Redis Pub/Sub"]
    Relay -->|"WEBHOOK_DISPATCH"| WQ["Webhook Queue"]

    AQ --> AW["assignmentWorker\nScores doctors by capacity\n+ recency + availability\nCAS UPDATE assignment"]
    NQ --> NW["notificationWorker\nDedup via dedupeKey\nMulti-channel delivery"]
    RQ --> RFW["refundWorker\nRazorpay refund API\nIdempotency key enforced"]
    RQ -.->|"Stuck recovery"| RRW["refundRecoveryWorker"]
    WQ --> WHW["webhookProcessorWorker\nIdempotent Razorpay events\nexternalEventId UNIQUE"]
    
    SW["sweeperWorker\n(scheduled)"] --> Sweepers
    
    subgraph Sweepers
        S1["paid-unassigned\ndetector"]
        S2["orphan-payment\ndetector"]
        S3["assignment-lease\nexpiry"]
        S4["outbox-stuck\nrecovery"]
        S5["payment-timeout\nenforcer"]
    end
```

**OutboxRelay Two-Phase Claim:**
```
Phase 1 — Atomic Claim:
  SELECT id FROM OutboxEvent
  WHERE status='PENDING' AND (nextRetryAt IS NULL OR nextRetryAt <= NOW())
  LIMIT 50
  FOR UPDATE SKIP LOCKED   ← Zero deadlock multi-process safe

Phase 2 — Process & Settle:
  Mark PROCESSING → execute → mark DISPATCHED
  On failure: exponential backoff (2^n × 2s), max 5 retries → FAILED_PERMANENT
```

---

## 🔒 Security Model

```mermaid
flowchart TD
    Request["Incoming HTTP Request"] --> MW

    MW["Layer 1: Next.js Middleware\n• CSRF cookie === x-csrf-token header\n• x-method header injection\n• x-correlation-id generation"]

    MW --> RBAC["Layer 2: RBAC Guards (lib/rbac.ts)\n• requireAuth() — session lookup\n• requireRole() — exact match\n• requireVerifiedDoctor()\n• canAccessConsultation()"]

    RBAC --> AppLayer["Layer 3: Application Logic\n• SQL CAS for state transitions\n• Idempotency keys on all finances\n• Optimistic version locks (consultation.version)"]

    AppLayer --> ORM["Layer 4: Prisma Extension\n• AuditLog update/delete → throws at ORM level\n• Slow query detection (>500ms)\n• Graceful SIGTERM shutdown"]
```

**HIPAA-Aligned Audit Logging:**
- Only `fieldNamesChanged` is logged (e.g., `["status", "startedAt"]`) — never actual medical values
- No PHI (Protected Health Information) ever touches audit tables
- `AuditLog` is structurally immutable — blocked at the ORM layer, not just policy

**Financial Idempotency:**
- `WebhookEvent.externalEventId` — `UNIQUE` constraint prevents duplicate payment processing
- `RefundRequest.idempotencyKey` — `UNIQUE` constraint prevents duplicate refunds
- `Notification.dedupeKey` — `UNIQUE(userId, dedupeKey)` prevents duplicate alerts

---

## 📡 Observability & Metrics

```mermaid
flowchart LR
    subgraph "Metrics Layers"
        HTTP["HTTP Layer\nhttp_requests_total\nhttp_request_duration_seconds\n{method · route · status}"]
        Socket["Socket Layer\nsockets.active · connects\ndisconnects · reconnects\nfanout_latency_ms"]
        Redis["Redis Health\npubsub_reconnects\nsubscriber_lag_ms\ndisconnects"]
        System["System Health\npaid_unassigned_count\noutbox_pending_count\nqueue_depth · anomalies"]
    end

    All["All metrics exposed at\n/api/metrics\n(Prometheus-compatible)"] 
    HTTP & Socket & Redis & System --> All
```

Every API request is wrapped with `withTrace()` which:
- Generates or propagates `x-correlation-id` through `AsyncLocalStorage`
- Attaches `x-trace-id` to every response for client-side debugging
- Records Prometheus request counter + duration histogram
- Triggers `sendAlert('CRITICAL', ...)` on any unhandled 500 error

All logs emitted as **structured Pino JSON** with `traceId`, `userId`, `event`, `durationMs` — ready for Loki, Datadog, or CloudWatch.

---

## 🗄 Database Schema

28 models, 18 enums — source of truth at [`prisma/schema.prisma`](prisma/schema.prisma).

```mermaid
erDiagram
    User ||--o{ ConsultationSession : "patient"
    User ||--o{ ConsultationSession : "doctor"
    User ||--|| DoctorProfile : "profile"
    User ||--o{ Session : "sessions"
    User ||--o{ AuditLog : "logs"

    ConsultationSession ||--o{ ChatMessage : "messages"
    ConsultationSession ||--o{ ConsultationEvent : "timeline"
    ConsultationSession ||--o{ Prescription : "prescriptions"
    ConsultationSession ||--o{ DiagnosticTest : "tests"
    ConsultationSession ||--o{ MedicalReport : "reports"
    ConsultationSession ||--o{ PaymentRecord : "payments"
    ConsultationSession ||--o{ RefundRequest : "refunds"
    ConsultationSession ||--o{ AssignmentAttempt : "attempts"
    ConsultationSession ||--o| ConsultationSnapshot : "snapshot"
    ConsultationSession ||--o| PatientConsent : "consent"

    PaymentRecord ||--o{ RefundRecord : "executions"
    RefundRequest ||--o{ RefundRecord : "records"

    OutboxEvent {
        string id PK
        string eventType
        string payload
        string status
        int retryCount
        string nextRetryAt
    }

    WebhookEvent {
        string externalEventId UK
        string status
    }
```

**Key Design Decisions in the Schema:**

| Pattern | Where Used |
|---|---|
| `version Int` optimistic lock | `ConsultationSession` — stale writes rejected |
| `idempotencyKey String UNIQUE` | `RefundRequest`, `RefundRecord` — no double refunds |
| `externalEventId String UNIQUE` | `WebhookEvent` — idempotent webhook dedup |
| `dedupeKey UNIQUE(userId, dedupeKey)` | `Notification` — no duplicate alerts |
| `FOR UPDATE SKIP LOCKED` | `OutboxEvent` — deadlock-free multi-process claim |
| Composite indices | `DoctorProfile(verificationStatus, acceptingNewPatients, availabilityStatus, lastSeenAt)` |

---

## 📡 API Reference

All mutations enforce CSRF at the middleware layer before route handlers run.

| Group | Endpoints | Highlights |
|---|---|---|
| **Auth** | `register · login · logout · me · realtime-ticket` | HTTP-only cookie sessions, single-use WS tickets |
| **Consultations** | `create · get · start · complete · cancel · message · prescription · tests` | CAS state transitions, idempotent retries |
| **Payments** | `create-order · webhook · mock` | HMAC-verified Razorpay webhooks, idempotent processing |
| **Reports** | `upload · download · verify` | SHA-256 hash verification, MIME validation |
| **Prescriptions** | `issue · verify/[code]` | Versioned prescriptions, public verification endpoint |
| **Doctor** | `queue · accept · heartbeat · availability` | Priority-sorted queue, atomic CAS accept |
| **Admin** | `stats · queue · assign · audit · export · verify-doctor` | KPI dashboard, CSV export, audit viewer |
| **Metrics / Health** | `/api/metrics · /api/health · /health/realtime` | Prometheus scrape, app + gateway health |

---

## 🛠 Tech Stack

| Category | Technology |
|---|---|
| **Framework** | Next.js 15.5 (App Router, Turbopack) |
| **Language** | TypeScript 5.7 (strict mode, across all layers) |
| **Database** | PostgreSQL 16 (ACID, CAS patterns, composite indices) |
| **ORM** | Prisma 6 with custom client extension (audit enforcement) |
| **Cache / Queue Bus** | Redis 7 via ioredis (sessions, rate limits, pub/sub, BullMQ) |
| **Job Queue** | BullMQ (8 named queues, exponential backoff, deduplication) |
| **WebSocket** | `ws` library (stateless gateway, heartbeat, backpressure) |
| **Schema Validation** | Zod (realtime event envelopes, API request payloads) |
| **Payments** | Razorpay SDK (payment + refund + HMAC webhook verification) |
| **File Storage** | AWS S3 SDK / Local disk (pluggable `StorageProvider`) |
| **Metrics** | prom-client (counters, histograms, gauges — low-cardinality enforced) |
| **Logging** | Pino (structured JSON, `AsyncLocalStorage` trace propagation) |
| **Testing** | Jest + ts-jest, Playwright (E2E) |
| **Chaos Testing** | Shopify Toxiproxy (network fault injection) |
| **Containerization** | Docker + Docker Compose (7-service chaos environment) |
| **CSS** | Tailwind CSS 4 |

---

## 📁 Project Structure

```
doctor-consult/
├── app/                          # Next.js App Router
│   ├── api/                      # 18 REST API route groups
│   │   ├── consultations/[id]/   # Lifecycle: start, complete, cancel, chat, prescription, tests
│   │   ├── payments/             # create-order, HMAC webhook, mock
│   │   ├── admin/                # stats, queue, assign, audit, export, verify-doctor
│   │   ├── auth/                 # register, login, logout, me, realtime-ticket
│   │   ├── doctor/               # queue, accept, heartbeat, availability
│   │   ├── reports/              # upload + authenticated download
│   │   ├── prescriptions/verify/ # Public prescription verification
│   │   ├── metrics/              # Prometheus scrape endpoint
│   │   └── health/               # App health check
│   ├── admin/                    # Admin dashboard UI
│   ├── doctor/                   # Doctor dashboard + history + profile
│   ├── patient/                  # Patient dashboard + history + profile
│   └── login/ register/ verify/  # Auth pages
│
├── realtime-gateway/             # Standalone WebSocket server :3001
│   ├── server.ts                 # HTTP upgrade · Auth · Redis Pub/Sub · DEGRADED mode
│   └── ConnectionManager.ts      # userId→socketId registry · heartbeat · 512KB backpressure
│
├── shared/realtime/
│   └── events.ts                 # Zod schemas: RealtimeEventSchema, ControlMessageSchema
│
├── worker/                       # Standalone background process
│   ├── main.ts                   # Bootstrap: queues, relay, sweepers, heartbeat
│   ├── outboxRelay.ts            # Two-phase SKIP LOCKED batch poll
│   ├── sweepers.ts               # Anomaly detection sweepers (5 sweepers)
│   ├── definitions/              # 8 BullMQ worker definitions
│   └── services/                 # heartbeat · notification providers · repeatables
│
├── lib/                          # Core server-side library
│   ├── consultation-state-machine.ts  # Pure function state machine
│   ├── rbac.ts                   # requireAuth · requireRole · canAccessConsultation
│   ├── auth.ts                   # Session: create, get (Redis-cached 30min), destroy
│   ├── prisma.ts                 # Singleton + AuditLog immutability extension
│   ├── redis-factory.ts          # Real/mock factory + placeholder-aware TLS detection
│   ├── queue.ts                  # 8 BullMQ Queue instances
│   ├── metrics.ts                # Prometheus MetricsRegistry (strict low-cardinality)
│   ├── withTrace.ts              # AsyncLocalStorage traceId + Prometheus recording
│   ├── lock.ts                   # Redis SET NX PX + Lua compare-and-delete release
│   ├── storage/                  # StorageProvider interface + S3 + Local implementations
│   └── audit.ts                  # Append-only audit log helpers
│
├── prisma/
│   ├── schema.prisma             # 28 models, 18 enums, 645 lines
│   └── seed.ts                   # Test data: patients, doctors, admins
│
├── tests/                        # 5 test profiles, 30+ scenarios
│   ├── api/                      # API integration (auth, lifecycle, payments, chat, uploads)
│   ├── reliability/              # Reconnect semantics, Postgres/Redis restart, SIGKILL recovery
│   ├── destructive/              # Chaos: CAS integrity, IDOR, race conditions, Docker crashes
│   ├── stress/                   # Throughput, concurrency, real-time chaos
│   └── contracts/                # Schema and invariant contracts
│
├── middleware.ts                  # CSRF + x-method + correlation-id (Edge)
├── toxiproxy.json                 # Toxiproxy: Redis :6379 + Postgres :5432 proxies
├── docker-compose.test.yml        # 7-service chaos environment
├── SYSTEM_INVARIANTS.md          # Non-negotiable architectural guarantees
└── AGENT_INSTRUCTIONS.md         # Instructions for AI coding agents on this repo
```

---

## 🧠 Key Engineering Decisions

### 1. SQL-Level CAS — Zero Application Locks
Every state transition uses `UPDATE ... WHERE status = 'EXPECTED' RETURNING id`. If 0 rows are updated, a concurrent actor already won. No Redis locks, no distributed coordination overhead, no failure modes from lock expiry.

### 2. `FOR UPDATE SKIP LOCKED` — Deadlock-Free Outbox
The OutboxRelay claims event batches using PostgreSQL's `SKIP LOCKED` clause. Multiple worker instances can run concurrently without ever deadlocking or processing the same event twice.

### 3. Pure-Function State Machine
`transition(session, params)` has zero side effects. It validates the transition and returns Prisma payload arrays. The caller commits everything in a single `$transaction`. This means the state machine is fully unit-testable without any infrastructure.

### 4. Backpressure in `finally` Blocks
Redis `INCR`/`DECR` concurrency counters are decremented in `finally` blocks — guaranteed execution even when `requireRole()` throws `ForbiddenError`. No permanent counter inflation from handler crashes.

### 5. ORM-Level Immutability
Prisma client extension intercepts all `update`/`delete` on `AuditLog` and throws a compliance error. This is enforced at the ORM layer — no policy document can override it at runtime.

### 6. Stateless Gateway via Redis Pub/Sub
The WebSocket gateway holds zero state about which consultations exist. It only manages socket connections. All domain events arrive from Redis. This allows the gateway to crash and restart without losing any domain state.

### 7. Versioned Events — Stale-Event Protection
`ConsultationStatusChanged` events carry `consultationVersion` (monotonic DB integer). Clients discard events with version ≤ their last-seen version, preventing stale events from corrupting UI state after reconnects.

### 8. Placeholder-Aware S3 Detection
```typescript
function isS3Configured(): boolean {
    const keyId = process.env.AWS_ACCESS_KEY_ID ?? '';
    if (!keyId || keyId.startsWith('your-')) return false;
    return true;
}
```
Prevents accidental AWS calls when `.env` contains placeholder values — a common cause of mysterious 500 errors in dev.

---

## 📜 System Invariants

These are **non-negotiable architectural guarantees** documented in `SYSTEM_INVARIANTS.md`. Every future change must prove it preserves all of them.

```mermaid
mindmap
  root((System Invariants))
    Financial
      One Payment → One Refund Maximum
      All refunds carry deterministic idempotency key
    Consultation
      One patient → one active consultation at a time
      One doctor per consultation max
      Post-completion prescriptions are immutable
    Messaging
      Persisted messages never disappear from broadcast
      At-least-once delivery via outbox relay
    Outbox
      Side effects committed in same tx as state change
      Events only marked processed after downstream ack
    Workers
      One queue → one worker implementation
      Unknown job types must fail loudly, not silently
      No queue nesting
    Redis
      maxmemory-policy must be noeviction
      Silent key eviction is forbidden
    Bootstrap
      Only main.ts may start infrastructure
      Imports must never start timers or connections
```

---

## 🧪 Testing & Chaos Engineering

The platform is proven through **five independent test profiles** in a fully Dockerized chaos environment.

```mermaid
graph LR
    subgraph "7-Service Chaos Environment (docker-compose.test.yml)"
        PG_T["PostgreSQL :5433"]
        REDIS_T["Redis :6380"]
        TOXY["Toxiproxy\n:8474 (control)\n:6379 (Redis proxy)\n:5432 (PG proxy)"]
        APP["app_chaos\n(full Next.js build)"]
        GW["realtime_gateway_chaos"]
        WK["worker_chaos"]
        RUNNER["test_runner\n(Jest executor)"]
    end

    PG_T & REDIS_T --> TOXY
    TOXY --> APP & GW & WK
    RUNNER --> APP
    RUNNER --> TOXY
```

> All services connect **through Toxiproxy**, not directly to Redis or Postgres. This allows tests to inject failures with a single HTTP call to the Toxiproxy control API.

### Test Profiles

| Profile | Command | What Is Tested |
|---|---|---|
| **Unit** | `npm run test:unit` | Pure logic functions, state machine transitions, validation |
| **API Contracts** | `npm run test:contracts` | Auth, RBAC enforcement, consultation lifecycle HTTP APIs |
| **Reliability** | `npm run test:reliability:docker` | WebSocket reconnect, Postgres/Redis restart, worker SIGKILL, pub/sub outage |
| **Destructive** | `npm run test:destructive:docker` | CAS race conditions, IDOR attacks, BullMQ stalled jobs, network chaos, Docker crashes |
| **Stress** | `npm run test:stress:docker` | API throughput, WebSocket concurrency, sustained chaos under load |

### What Is Proven

**Race Condition Safety**
50 concurrent doctors attempt to accept the same patient simultaneously over a Toxiproxy-jittered network. The CAS guarantee ensures exactly one succeeds every time.

**Outbox Durability**
Redis is severed mid-dispatch. After reconnection, the OutboxRelay detects the stuck `PROCESSING` event and replays it — zero message loss.

**SIGKILL Recovery**
The BullMQ worker container is forcibly killed mid-job execution. On restart, BullMQ's stall detection re-queues the job and the worker processes it cleanly — no orphaned locks, no corrupted data.

**Financial Integrity**
The Grand Orchestrator runs after all chaos tests complete and executes mathematical invariant checks directly against PostgreSQL:
- No COMPLETED payment against a CANCELLED session
- No session in PAID/READY/ACTIVE without a COMPLETED PaymentRecord
- No active session without a doctorId
- No doctor assigned to more consultations than their `maxActiveCases`
- No OutboxEvent stuck PENDING for more than 2 minutes

**Security**
IDOR (Insecure Direct Object Reference) attacks tested: accessing another patient's consultation, another doctor's prescription, admin endpoints without role — all return 403 with no data leakage.

---

## 🚀 Quick Start

### Prerequisites
- Node.js 20+, Docker Desktop, npm

```bash
# 1. Clone and install
git clone <repo> && cd doctor-consult && npm install

# 2. Configure environment
cp .env.example .env  # Fill in DATABASE_URL, REDIS_HOST, SESSION_SECRET, RAZORPAY keys

# 3. Start infrastructure
docker compose up -d

# 4. Initialise database
npm run prisma:generate && npm run prisma:migrate && npm run seed

# 5. Start all services
npm run dev:all
# → Next.js API on :3000
# → WebSocket Gateway on :3001
# → Background Worker
```

---

## ☁️ Deployment

### Architecture Overview

```mermaid
graph LR
    subgraph "Production Infrastructure"
        LB["Load Balancer"]
        subgraph "App Containers (scale ×N)"
            APP1["Next.js :3000"]
            APP2["Next.js :3000"]
        end
        subgraph "Gateway Containers (scale ×M)"
            GW1["WS Gateway :3001"]
            GW2["WS Gateway :3001"]
        end
        subgraph "Worker Containers (scale ×K)"
            WK1["BullMQ Worker"]
        end
        PG[("PostgreSQL\n+ PgBouncer pooler")]
        REDIS_PROD[("Redis (TLS)\nUpstash / ElastiCache")]
        S3_PROD["AWS S3"]
        RAZORPAY["Razorpay"]
    end

    LB --> APP1 & APP2 & GW1 & GW2
    APP1 & APP2 & GW1 & GW2 & WK1 --> PG & REDIS_PROD
    APP1 & APP2 --> S3_PROD & RAZORPAY
```

### Production Checklist

| Item | Detail |
|---|---|
| `NODE_ENV=production` | Required |
| `DATABASE_URL` | Via PgBouncer or Neon for connection pooling |
| `DIRECT_URL` | Primary DB URL for Prisma migrations |
| `REDIS_TLS=true` | Required for Upstash / ElastiCache |
| `RAZORPAY_WEBHOOK_SECRET` | Required for HMAC webhook verification |
| `ENABLE_MOCK_PAYMENT=false` | Must be explicitly disabled in production |
| `SESSION_SECRET` | 32+ random characters |
| Redis `maxmemory-policy noeviction` | **System invariant — mandatory** |
| Worker as separate container | `npm run worker` |
| Gateway as separate container | `npm run realtime` |
| Persistent volume for `uploads/` | If using local disk fallback for files |

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | ✅ | PostgreSQL (via pooler in prod) |
| `DIRECT_URL` | ✅ | Direct PostgreSQL (migrations) |
| `REDIS_URL` | ✅ | Redis connection string |
| `SESSION_SECRET` | ✅ | 32+ char random string |
| `RAZORPAY_KEY_ID` | ✅ | Razorpay public key |
| `RAZORPAY_KEY_SECRET` | ✅ | Razorpay secret key |
| `RAZORPAY_WEBHOOK_SECRET` | Prod | HMAC webhook verification |
| `ENABLE_MOCK_PAYMENT` | Dev | `true` to bypass Razorpay in dev |
| `AWS_ACCESS_KEY_ID` | S3 | Leave blank for local disk fallback |
| `AWS_SECRET_ACCESS_KEY` | S3 | Leave blank for local disk fallback |
| `S3_BUCKET_NAME` | S3 | Bucket name |
| `REDIS_TLS` | Cloud | `true` for Upstash / ElastiCache |
| `LOG_LEVEL` | Optional | `info` / `debug` / `error` |
| `REALTIME_PORT` | Optional | Gateway port (default: 3001) |

---

## 📄 License

MIT License © 2026 — Built for enterprise telemedicine infrastructure.

---

<div align="center">

**Built with precision. Designed for compliance. Proven under chaos. Ready for production.**

</div>
