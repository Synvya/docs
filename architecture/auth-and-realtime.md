# Auth & Real-Time Event Processing — System Architecture

System-wide architecture for Nostr key custody, flexible authentication, and 24/7 event processing across Synvya services. This document defines how the components connect, the API contracts between them, and the deployment topology. Each component's implementation spec lives in its own repository.

## 1. Problem Statement

Synvya's current architecture has two structural limitations:

1. **No 24/7 operation**: Nostr private keys live only in the restaurant owner's browser (IndexedDB). When the browser is closed, no events can be signed — reservation responses, profile updates, and offer publications all stop.
2. **No flexible authentication**: Users must manage Nostr keypairs directly. There is no email/password option and no path to social login. This limits adoption to Nostr-native users.
3. **Nostr protocol leaks into the wrong layer**: The MCP server — whose job is providing an AI-friendly interface — contains Nostr protocol details (NIP-59 gift wrapping, relay WebSocket management, private key handling). This makes both the Nostr logic and the AI interface harder to maintain.

## 2. Design Principles

**Nostr-first**: Nostr events on relays are the source of truth. All other data stores (DynamoDB, PostgreSQL) are caches or operational state. Every business action (publish profile, respond to reservation) ultimately produces a Nostr event.

**Single Nostr boundary**: Exactly one service — the Event Processor — interacts with Nostr relays. All other services speak business-domain language (restaurants, reservations, menus) through internal APIs. No Nostr concepts (pubkeys, event kinds, gift wraps) leak into the MCP server or other consumer-facing interfaces.

**Honest async**: Nostr is inherently asynchronous (publish event, wait for response on relays). Rather than hiding this behind fragile polling loops, expose it as a two-step pattern: submit a request, check its status later. AI agents handle multi-step workflows well.

**Key separation**: No service other than Keycast ever holds or sees private keys. The Event Processor and MCP server request signatures via Keycast's HTTP RPC.

## 3. Target Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            AI Agents                                    │
│                     (Claude, ChatGPT, etc.)                             │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ MCP protocol / OpenAPI
                               ▼
                    ┌──────────────────────┐
                    │  MCP Server          │
                    │  (Vercel)            │
                    │                      │
                    │  • AI-friendly API   │
                    │  • Zero Nostr code   │
                    │  • Thin translation  │
                    │    layer             │
                    └──────────┬───────────┘
                               │ Internal HTTP API
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  EC2 Instance                                                           │
│                                                                         │
│  ┌───────────────────────────┐    ┌──────────────────────────────────┐  │
│  │  Event Processor          │    │  Keycast                         │  │
│  │  (Node.js)                │    │  (Rust)                          │  │
│  │                           │    │                                  │  │
│  │  • Sole Nostr interface   │◄──►│  • Key custody (AES-256-GCM)    │  │
│  │  • Persistent relay conns │    │  • Auth (email/pw, Nostr key)   │  │
│  │  • NIP-59, NIP-RP logic   │    │  • Signing RPC (~50ms)          │  │
│  │  • Internal HTTP API      │    │  • NIP-46 bunker                │  │
│  │  • Event handlers         │    │                                  │  │
│  └─────────┬─────────────────┘    └──────────┬───────────────────────┘  │
│            │                                  │                         │
│            │                                  │                         │
└────────────┼──────────────────────────────────┼─────────────────────────┘
             │                                  │
     ┌───────┴────────┐               ┌────────┴─────────┐
     │  Nostr Relays   │               │  RDS PostgreSQL  │
     │  (damus, nos.   │               │  ElastiCache     │
     │   lol, snort)   │               │  AWS KMS         │
     └────────────────┘               └──────────────────┘
             │
             │ WebSocket (persistent)
             ▼
     ┌────────────────┐
     │  DynamoDB       │
     │  (event cache   │
     │   + reservation │
     │   state)        │
     └────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  Browser                                                                │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Synvya Client (React)                                           │   │
│  │  • Login via Keycast (email/pw or Nostr key)                     │   │
│  │  • Signs events via Keycast HTTP RPC                             │   │
│  │  • Publishes events via relay connections (when online)          │   │
│  │  • No private keys in browser                                    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## 4. Component Responsibilities

| Component | Repository | Responsibility | Nostr Knowledge |
|---|---|---|---|
| **Keycast** | `Synvya/keycast` | Key custody, authentication, event signing | Signing only (no relay communication) |
| **Event Processor** | `Synvya/event-processor` | Relay connections, event routing, NIP-59/NIP-RP protocol, internal API | Full (sole relay interface) |
| **MCP Server** | `Synvya/mcp-server` | AI-agent-friendly tool interface, translates tool calls to Event Processor API | None |
| **Client** | `Synvya/client` | Restaurant owner UI, login, profile/menu/offer editing | Minimal (relay publishing when online, event format construction) |

## 5. Keycast Infrastructure

Self-hosted fork of [divinevideo/keycast](https://github.com/divinevideo/keycast) adapted for AWS.

### 5.1 AWS Resources

| Resource | Type | Purpose |
|---|---|---|
| EC2 | `t3.medium` | Keycast unified binary (API + NIP-46 signer) + Event Processor |
| RDS PostgreSQL | `db.t3.micro` | Users, encrypted keys, OAuth apps, policies |
| ElastiCache Redis | `cache.t3.micro` | Cluster coordination (hashring) |
| AWS KMS | Symmetric key | Master encryption key for AES-256-GCM key wrapping |
| ALB | Application LB | TLS termination, sticky sessions for HTTP RPC cache locality |
| ACM | Certificate | TLS for `auth.synvya.com` |
| SES | Email service | Verification emails, password resets |

### 5.2 Required Keycast Modifications

Two changes from upstream Keycast, isolated behind provider interfaces:

**1. KMS Provider**: Replace `google-cloud-kms` with `aws-sdk-kms` in `core/src/encryption.rs`. New `AwsKmsProvider` implementing the same trait. Selected via `KMS_PROVIDER=aws|gcp|file`.

**2. Email Provider**: Replace SendGrid with Amazon SES in `api/src/api/http/auth.rs`. New `AwsSesEmailProvider`. Selected via `EMAIL_PROVIDER=ses|sendgrid|console`.

### 5.3 Authentication Methods

**v1**:
- **Email/password**: `POST /api/auth/register` and `/login`. Bcrypt hashing. Keycast generates Nostr keypair on registration. UCAN session tokens (24h expiry).
- **Nostr key import**: `POST /api/auth/register` with nsec. Keycast encrypts and stores the key.

**v2** (architecture prepared, not implemented):
- **Google/Facebook**: Auth gateway (AWS Cognito or custom) handles social OAuth, provisions/links Nostr keys in Keycast. No Keycast code changes needed — social identities map to standard Keycast user accounts.

### 5.4 Signing Policy

A "Synvya Restaurant" policy restricts signing to permitted event kinds:

| Kind | Purpose |
|---|---|
| 0 | Business profile |
| 1059 | Gift wrap (NIP-59) |
| 9901 | Reservation request |
| 9902 | Reservation response |
| 9903 | Modification request |
| 9904 | Modification response |
| 30402 | Menu item |
| 30405 | Menu collection |
| 31556 | Offer/promotion |

### 5.5 Network Security

| Resource | Inbound Rules |
|---|---|
| ALB | 443 (HTTPS) from `0.0.0.0/0` |
| EC2 | 3000, 5173 from ALB SG. 22 from admin IP. |
| RDS | 5432 from EC2 SG only |
| ElastiCache | 6379 from EC2 SG only |

### 5.6 Deployment

Docker Compose on EC2. Image built from `Synvya/keycast` fork, pushed to ECR. CI/CD via GitHub Actions: build → push to ECR → SSH pull + restart.

Domain: `auth.synvya.com` (production), `auth-staging.synvya.com` (staging, same EC2, separate Keycast tenant via domain-based isolation).

## 6. Internal API Contract

The Event Processor exposes an internal HTTP API that the MCP server and other consumers call. This API uses business-domain language — no Nostr concepts.

**Base URL**: `http://localhost:4000` (same EC2 as Keycast; not publicly exposed)

For external access from the MCP server (Vercel), the ALB routes `/api/events/*` to the Event Processor on port 4000. The MCP server calls `https://auth.synvya.com/api/events/*`.

### 6.1 Reservations

**Create a reservation request**

```
POST /api/events/reservations
Content-Type: application/json
Authorization: Bearer <keycast-service-token>

{
  "restaurant_id": "npub1abc...",
  "customer_id": "npub1xyz...",
  "party_size": 4,
  "time": "2026-04-01T19:00:00-07:00",
  "notes": "Window seat preferred"
}

Response 202 Accepted:
{
  "reservation_id": "abc123...",
  "status": "pending",
  "created_at": "2026-03-30T15:00:00Z"
}
```

The Event Processor:
1. Builds a kind 9901 rumor from the business-domain fields
2. Signs it via Keycast HTTP RPC
3. Wraps in NIP-59 (seal + gift wrap, each signed/encrypted via Keycast)
4. Publishes to relays
5. Stores pending state in DynamoDB
6. Returns immediately with `202 Accepted`

**Check reservation status**

```
GET /api/events/reservations/:reservation_id
Authorization: Bearer <keycast-service-token>

Response 200:
{
  "reservation_id": "abc123...",
  "status": "confirmed",
  "restaurant_id": "npub1abc...",
  "customer_id": "npub1xyz...",
  "party_size": 4,
  "time": "2026-04-01T19:00:00-07:00",
  "responded_at": "2026-03-30T15:02:30Z"
}
```

Reads from DynamoDB. Status values: `pending`, `confirmed`, `declined`, `cancelled`, `modification_requested`, `modification_confirmed`, `modification_declined`.

**List reservations for a restaurant**

```
GET /api/events/reservations?restaurant_id=npub1abc...&status=pending
Authorization: Bearer <keycast-service-token>

Response 200:
{
  "reservations": [
    { "reservation_id": "abc123...", "status": "pending", ... },
    { "reservation_id": "def456...", "status": "pending", ... }
  ]
}
```

**Respond to a reservation**

```
POST /api/events/reservations/:reservation_id/respond
Content-Type: application/json
Authorization: Bearer <keycast-service-token>

{
  "status": "confirmed"
}

Response 200:
{
  "reservation_id": "abc123...",
  "status": "confirmed",
  "responded_at": "2026-03-30T15:02:30Z"
}
```

The Event Processor:
1. Builds a kind 9902 rumor with the response status and thread reference
2. Signs via Keycast
3. Wraps in NIP-59, publishes to relays
4. Updates DynamoDB

### 6.2 Discovery Data (Read-Only)

The Event Processor also serves discovery data that it collects from relays (replacing the Lambda `nostr-querier` long-term):

```
GET /api/events/restaurants
GET /api/events/restaurants/:id/menu
GET /api/events/restaurants/:id/offers
```

**v1**: These endpoints are NOT implemented. The MCP server continues to read discovery data from DynamoDB (populated by the existing Lambda). The Event Processor focuses on reservation flows only.

**v2**: The Event Processor subscribes to discovery event kinds (0, 30402, 30405, 31556) and serves them via these endpoints, replacing the Lambda.

### 6.3 Authentication

All internal API calls require a `Bearer` token. Two token types:

| Consumer | Token Source | Scope |
|---|---|---|
| MCP Server | Keycast OAuth client credentials | Read/write reservations, read discovery data |
| Client App | User's Keycast session token (passed through) | Read/write for own restaurant only |

The Event Processor validates tokens by calling Keycast's token introspection endpoint.

## 7. Data Flow: Reservation Lifecycle

### 7.1 Customer Makes a Reservation (via AI Agent)

```
AI Agent                MCP Server              Event Processor         Keycast          Relays
   │                       │                         │                    │                │
   │  make_reservation     │                         │                    │                │
   │  (restaurant, time,   │                         │                    │                │
   │   party_size)         │                         │                    │                │
   │──────────────────────>│                         │                    │                │
   │                       │  POST /reservations     │                    │                │
   │                       │  { restaurant_id,       │                    │                │
   │                       │    party_size, time }   │                    │                │
   │                       │────────────────────────>│                    │                │
   │                       │                         │  sign_event(9901)  │                │
   │                       │                         │───────────────────>│                │
   │                       │                         │  signed event      │                │
   │                       │                         │<───────────────────│                │
   │                       │                         │  nip44_encrypt     │                │
   │                       │                         │───────────────────>│                │
   │                       │                         │  ciphertext        │                │
   │                       │                         │<───────────────────│                │
   │                       │                         │                    │  publish 1059  │
   │                       │                         │───────────────────────────────────>│
   │                       │                         │  write DynamoDB    │                │
   │                       │  202 { id, "pending" }  │                    │                │
   │                       │<────────────────────────│                    │                │
   │  { reservation_id,   │                         │                    │                │
   │    status: "pending" }│                         │                    │                │
   │<──────────────────────│                         │                    │                │
   │                       │                         │                    │                │
   │  check_reservation   │                         │                    │                │
   │  (reservation_id)     │                         │                    │                │
   │──────────────────────>│  GET /reservations/:id  │                    │                │
   │                       │────────────────────────>│                    │                │
   │                       │  { status: "confirmed" }│                    │                │
   │                       │<────────────────────────│                    │                │
   │  { status:            │                         │                    │                │
   │    "confirmed" }      │                         │                    │                │
   │<──────────────────────│                         │                    │                │
```

### 7.2 Restaurant Responds (via Client App)

```
Client App              Event Processor         Keycast          Relays
   │                         │                    │                │
   │  GET /reservations      │                    │                │
   │  ?status=pending        │                    │                │
   │────────────────────────>│                    │                │
   │  [pending reservations] │                    │                │
   │<────────────────────────│                    │                │
   │                         │                    │                │
   │  POST /reservations/    │                    │                │
   │  :id/respond            │                    │                │
   │  { status: "confirmed" }│                    │                │
   │────────────────────────>│                    │                │
   │                         │  sign_event(9902)  │                │
   │                         │───────────────────>│                │
   │                         │  nip44_encrypt     │                │
   │                         │───────────────────>│                │
   │                         │                    │  publish 1059  │
   │                         │───────────────────────────────────>│
   │                         │  update DynamoDB   │                │
   │  { status: "confirmed" }│                    │                │
   │<────────────────────────│                    │                │
```

### 7.3 Restaurant Receives Reservation While Offline

```
Relays              Event Processor         Keycast          DynamoDB
   │                     │                    │                │
   │  kind 1059          │                    │                │
   │  (gift wrap)        │                    │                │
   │────────────────────>│                    │                │
   │                     │  nip44_decrypt     │                │
   │                     │───────────────────>│                │
   │                     │  plaintext         │                │
   │                     │<───────────────────│                │
   │                     │  (inner: kind 9901)│                │
   │                     │                    │                │
   │                     │  write reservation │                │
   │                     │  status: "pending" │                │
   │                     │───────────────────────────────────>│
   │                     │                    │                │
   │  (later, owner opens client app)         │                │
   │  GET /reservations?status=pending        │                │
   │                     │───────────────────────────────────>│
   │                     │  [pending list]    │                │
   │                     │<───────────────────────────────────│
```

## 8. Implementation Repositories

| Repository | Status | Spec Location |
|---|---|---|
| `Synvya/keycast` | New (fork of `divinevideo/keycast`) | This document, Section 5 |
| `Synvya/event-processor` | New | `event-processor/docs/specs/event-processor.md` |
| `Synvya/mcp-server` | Existing | `mcp-server/docs/specs/thin-client-migration.md` |
| `Synvya/client` | Existing | `client/docs/specs/auth-migration.md` |
| `Synvya/docs` | New | This document |

### 8.1 Dependency Order

Implementation must proceed in this order (each step is a prerequisite for the next):

```
1. Synvya/keycast          ← Deploy first (auth + signing infrastructure)
     │
     ▼
2. Synvya/event-processor  ← Depends on Keycast for signing and decryption
     │
     ├──────────────────┐
     ▼                  ▼
3a. Synvya/mcp-server  3b. Synvya/client   ← Both depend on Event Processor API
    (can be parallel)      (can be parallel)
```

**Step 1**: Fork Keycast, apply AWS modifications (KMS, SES), deploy to EC2 with RDS/ElastiCache/KMS. Verify email/password registration and Nostr key import. Verify HTTP RPC signing.

**Step 2**: Build Event Processor. Deploy to same EC2. Verify persistent relay connections, gift wrap decryption via Keycast, DynamoDB writes, internal API endpoints.

**Step 3a**: Migrate MCP server to call Event Processor API. Remove Nostr dependencies. Add `check_reservation` tool.

**Step 3b**: Migrate client app to Keycast auth. Replace local signing with Keycast HTTP RPC. Add login page. Build migration flow for existing users.

## 9. Monitoring & Observability

| Service | Health Check | Key Metrics |
|---|---|---|
| Keycast | `GET /api/health` via ALB | Login latency, signing latency, active sessions |
| Event Processor | `GET /health` (internal) | Relay connections (gauge), events received/processed/failed, Keycast RPC latency |
| MCP Server | Vercel built-in | Tool call latency, error rate |
| Client | Browser analytics | Login success rate, migration completion rate |

CloudWatch alarms:
- Keycast unhealthy (ALB target) → Critical
- Event Processor relay connections < 1 for 5 min → Critical
- Event Processor events failed > 10 in 5 min → Warning
- Keycast signing latency P99 > 500ms → Warning

## 10. Security Summary

| Concern | Mitigation |
|---|---|
| Private key exposure | Keys encrypted at rest (AES-256-GCM + KMS). Decrypted only in Keycast memory. Never transmitted to other services. |
| Man-in-the-middle | TLS everywhere: ALB terminates HTTPS, internal traffic on localhost (same EC2). |
| Unauthorized signing | Policy-based authorization. Each OAuth app can only sign permitted event kinds. |
| Session hijack | UCAN tokens (24h expiry). Stored in localStorage (client) or Parameter Store (services). |
| Database breach | RDS encrypted at rest. Private subnet. Security group restricted to EC2 only. |
| Relay impersonation | Events verified by Nostr signature. Gift wraps validated per NIP-17 (seal pubkey = rumor pubkey). |

## 11. Future Enhancements (v2+)

- **Social login**: Google/Facebook via auth gateway (Cognito or custom) → Keycast account linking
- **Auto-response engine**: Business rules in Event Processor for automatic reservation confirmation based on availability
- **Discovery data serving**: Event Processor subscribes to kinds 0/30402/30405/31556, replaces Lambda `nostr-querier`
- **DynamoDB Streams**: Push reservation status changes to MCP server instead of polling
- **Horizontal scaling**: Second EC2 instance, Redis hashring distributes NIP-46 and Event Processor subscriptions
- **Takeout orders**: New event kind handlers in Event Processor when protocol is defined
- **Key export**: Allow restaurant owners to export nsec for self-custody
