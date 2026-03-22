# Architecture Overview

## System Architecture

Don Film w/ Me is a service-oriented monolith (modular monolith) built with FastAPI. Services are logically separated into modules but deployed as a single application for MVP simplicity, with clear boundaries that allow extraction into microservices later if needed.

```
┌─────────────────────────────────────────────────────────────┐
│                      Mobile App                             │
│              (React Native / Flutter TBD)                   │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTPS + WebSocket
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   API Gateway (FastAPI)                      │
│                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐  │
│  │   Auth   │ │ Discover │ │   Feed   │ │   Messaging   │  │
│  │  Module  │ │  Module  │ │  Module  │ │    Module     │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐  │
│  │ Profile  │ │ Matching │ │ Booking  │ │   Reporting   │  │
│  │ Module   │ │  Module  │ │  Module  │ │    Module     │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────┘  │
└────┬──────────────┬──────────────┬──────────────┬───────────┘
     │              │              │              │
     ▼              ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐
│PostgreSQL│  │  Redis   │  │  S3 +    │  │ Celery +      │
│+ PostGIS │  │          │  │  CDN     │  │ RabbitMQ      │
└──────────┘  └──────────┘  └──────────┘  └───────────────┘
```

## Core Modules

### Auth Module
- JWT-based authentication with refresh tokens
- OAuth2 social login (Google, Apple)
- Phone number verification via SMS OTP (Twilio)
- Token blacklisting on logout (stored in Redis with TTL)

### Profile Module
- User profile CRUD
- Portfolio management (media, credits, resume)
- Rate card and availability calendar
- **Mode toggle**: switches `active_mode` between `talent` and `client`

### Discover Module
- Generates swipe queues based on filters and scoring
- Serves batches of 20 profile cards
- Pre-caches next batch in Redis while user swipes current batch

### Matching Module
- Records swipes (like, pass, superlike)
- Detects mutual likes → creates Match
- Triggers push notification on new match

### Feed Module
- Public content feed with ranking algorithm
- Post creation (link existing media items to a feed post)
- Like, comment, share, follow

### Messaging Module
- Real-time WebSocket chat (post-match)
- Message persistence in PostgreSQL
- Read receipts, typing indicators via WebSocket events
- **Concern mitigation**: Messages only between matched users. Unmatched = chat deleted.

### Booking Module
- Project-based booking requests
- Can be sent to matches (free) or non-matches (rate-limited, requires detailed description)
- Status workflow: pending → accepted/declined → completed

### Reporting Module
- User and content reporting
- Automated escalation thresholds
- Admin review queue

## Mode System (Hybrid Talent/Client)

Each user has a single account with an `active_mode` toggle:

```
┌─────────────────────────────────────────────────┐
│                  User Account                    │
│                                                  │
│  ┌─────────────────┐  ┌──────────────────────┐  │
│  │   Talent Mode   │  │    Client Mode       │  │
│  │                 │  │                      │  │
│  │ • Portfolio     │  │ • Project briefs     │  │
│  │ • Reels/Photos  │  │ • Casting needs      │  │
│  │ • Credits       │  │ • Budget range       │  │
│  │ • Rates         │  │ • Past projects      │  │
│  │ • Availability  │  │ • Reviews given      │  │
│  │                 │  │                      │  │
│  │ Sees: Clients   │  │ Sees: Talent         │  │
│  └─────────────────┘  └──────────────────────┘  │
│                                                  │
│  Shared: name, avatar, bio, location, messages   │
└─────────────────────────────────────────────────┘
```

### Mode-Aware Behavior

| Feature | Talent Mode | Client Mode |
|---------|------------|-------------|
| Swipe queue shows | Clients with projects/casting calls | Talent with matching roles |
| Profile card displays | Reel, headshot, credits, rate range | Project description, company, budget |
| Feed content priority | Behind-the-scenes, audition tips | Show reels, finished work |
| Booking requests | Receives them | Sends them |
| Notifications | "New casting opportunity nearby" | "New actor matches your search" |

### Concern: Mode-Switching Abuse

**Risk**: A user could switch modes rapidly to game the system (e.g., see who liked them in the other mode).

**Mitigations**:
- Mode switch cooldown: minimum 1 hour between switches
- Swipe history is mode-aware — switching modes doesn't reset your swipe queue
- Likes/passes are recorded with the mode they were made in
- Rate limiting on mode switches: max 5 per 24 hours

## Data Flow: Core Swipe-to-Match Journey

```
1. User opens app in Client Mode
2. GET /discover/queue → API checks Redis cache for pre-built queue
   ├─ Cache hit → return cached cards
   └─ Cache miss → query PostgreSQL with filters → score & rank → cache in Redis → return
3. User swipes right on a talent profile
4. POST /discover/swipe {target_id, direction: "like"}
   ├─ Record swipe in PostgreSQL
   ├─ Check: has target already liked this user?
   │   ├─ Yes → Create Match → Push notification to both → return {matched: true}
   │   └─ No → return {matched: false}
5. On match: messaging channel opened automatically
6. Either user can send a booking request through the match
```

## Security Architecture

- All API endpoints require JWT authentication (except auth routes)
- Rate limiting per endpoint via Redis (sliding window)
- Request validation via Pydantic schemas
- SQL injection prevention via SQLAlchemy ORM (parameterized queries)
- File upload validation: MIME type checking, file size limits, virus scanning
- CORS restricted to mobile app origins
- HTTPS enforced at load balancer level

See [Trust & Safety](trust-safety.md) for detailed abuse prevention measures.
