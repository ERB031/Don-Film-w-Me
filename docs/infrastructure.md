# Infrastructure & Deployment

## System Components

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Cloud Provider (AWS)                         │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
│  │   ALB +     │  │  FastAPI    │  │  Celery     │                │
│  │   CloudFront│──│  App (ECS)  │──│  Workers    │                │
│  │   (CDN)     │  │  2+ tasks   │  │  (ECS)      │                │
│  └─────────────┘  └──────┬──────┘  └──────┬──────┘                │
│                          │                 │                        │
│  ┌───────────────────────┼─────────────────┼──────────────────────┐ │
│  │                       │    Data Layer   │                      │ │
│  │  ┌──────────┐  ┌──────┴─────┐  ┌───────┴────┐  ┌───────────┐ │ │
│  │  │PostgreSQL│  │   Redis    │  │ RabbitMQ   │  │    S3     │ │ │
│  │  │  (RDS)   │  │(ElastiCache│  │(Amazon MQ) │  │  (Media)  │ │ │
│  │  │+ PostGIS │  │  )         │  │            │  │           │ │ │
│  │  └──────────┘  └────────────┘  └────────────┘  └───────────┘ │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ MediaConvert │  │     SES      │  │    SNS       │             │
│  │ (Transcode)  │  │   (Email)    │  │(Push Notify) │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└──────────────────────────────────────────────────────────────────────┘
```

## FastAPI Application Structure

```
don-film-w-me/
├── app/
│   ├── __init__.py
│   ├── main.py                         # FastAPI app, middleware, startup/shutdown
│   ├── config.py                       # Pydantic Settings (env vars)
│   ├── database.py                     # Async SQLAlchemy engine, session factory
│   ├── dependencies.py                 # FastAPI Depends (get_db, get_current_user)
│   │
│   ├── models/                         # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── user.py                     # User, UserRole
│   │   ├── portfolio.py                # Portfolio, MediaItem
│   │   ├── availability.py            # AvailabilitySlot
│   │   ├── swipe.py                    # Swipe
│   │   ├── match.py                    # Match
│   │   ├── booking.py                  # BookingRequest
│   │   ├── feed.py                     # FeedPost, FeedLike, FeedComment
│   │   ├── follow.py                   # Follow
│   │   ├── message.py                  # Message
│   │   ├── notification.py            # Notification
│   │   └── report.py                   # Report, AdminAction
│   │
│   ├── schemas/                        # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── user.py
│   │   ├── portfolio.py
│   │   ├── discover.py
│   │   ├── match.py
│   │   ├── booking.py
│   │   ├── feed.py
│   │   ├── message.py
│   │   └── report.py
│   │
│   ├── api/                            # Route handlers (thin — delegate to services)
│   │   ├── __init__.py
│   │   ├── router.py                   # Main router aggregating all sub-routers
│   │   ├── auth.py
│   │   ├── users.py
│   │   ├── portfolio.py
│   │   ├── discover.py
│   │   ├── matches.py
│   │   ├── bookings.py
│   │   ├── feed.py
│   │   ├── messages.py
│   │   ├── availability.py
│   │   ├── reports.py
│   │   └── websocket.py               # WebSocket chat endpoint
│   │
│   ├── services/                       # Business logic (testable, framework-agnostic)
│   │   ├── __init__.py
│   │   ├── auth_service.py             # Registration, login, token management
│   │   ├── user_service.py             # Profile updates, mode switching
│   │   ├── matching_service.py         # Queue generation, scoring, match detection
│   │   ├── feed_service.py             # Feed ranking, content management
│   │   ├── media_service.py            # Upload handling, S3 interaction
│   │   ├── moderation_service.py       # Auto-moderation, flag handling
│   │   ├── notification_service.py     # Push notification dispatch
│   │   ├── booking_service.py          # Booking request workflow
│   │   └── trust_service.py            # Trust score calculation, spam detection
│   │
│   ├── tasks/                          # Celery async tasks
│   │   ├── __init__.py
│   │   ├── celery_app.py              # Celery configuration
│   │   ├── transcode.py               # Video transcoding pipeline
│   │   ├── moderation.py              # Async content moderation
│   │   ├── notifications.py           # Push notification delivery
│   │   ├── trust_score.py             # Periodic trust score recalculation
│   │   └── cleanup.py                 # Expired bookings, cold storage migration
│   │
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── rate_limiter.py            # Redis-backed sliding window rate limiting
│   │   ├── request_logging.py         # Structured request/response logging
│   │   └── cors.py                     # CORS configuration
│   │
│   └── utils/
│       ├── __init__.py
│       ├── jwt.py                      # JWT creation, validation, blacklisting
│       ├── geo.py                      # PostGIS helpers, distance calculations
│       ├── storage.py                  # S3 upload/download/presigned URLs
│       ├── sms.py                      # Twilio SMS OTP
│       └── hashing.py                  # Password hashing, perceptual hashing
│
├── migrations/                         # Alembic migrations
│   ├── env.py
│   ├── alembic.ini
│   └── versions/
│
├── tests/
│   ├── conftest.py                    # Fixtures, test database, test client
│   ├── test_auth.py
│   ├── test_discover.py
│   ├── test_matching.py
│   ├── test_feed.py
│   ├── test_bookings.py
│   └── test_moderation.py
│
├── docs/                               # This design documentation
│
├── scripts/
│   ├── seed_data.py                   # Development seed data
│   └── run_migrations.py
│
├── .env.example                        # Template for environment variables
├── .gitignore
├── Dockerfile
├── docker-compose.yml                  # Local dev: postgres, redis, rabbitmq, app
├── requirements.txt
├── requirements-dev.txt               # Test/lint dependencies
├── pyproject.toml                     # Project metadata, tool config
└── README.md
```

---

## Video Transcoding Pipeline

### The Concern
Video is the most expensive part of this platform — storage, transcoding, and bandwidth costs scale linearly with users.

### Architecture

```
User Upload → S3 (originals bucket)
                    │
                    ▼
           Celery Task: transcode_video
                    │
                    ├─ Extract thumbnail (FFmpeg, frame at 1s)
                    ├─ Transcode to 720p (H.264, AAC, MP4)
                    ├─ Transcode to 480p (for slow connections)
                    ├─ Generate preview GIF (3 seconds)
                    │
                    ▼
              S3 (processed bucket) → CloudFront CDN
```

### Cost Control Measures

| Strategy | Implementation | Estimated Savings |
|----------|---------------|-------------------|
| **Duration limit** | 60-second max for MVP | Bounds per-video cost |
| **Upload size limit** | 100MB max | Prevents massive files |
| **Lazy 1080p** | Only transcode to 1080p when a video gets 50+ views | 40-60% transcoding cost reduction |
| **CDN caching** | CloudFront with 24h TTL for processed videos | Reduces S3 egress |
| **Cold storage** | Media not viewed in 180 days → S3 Glacier | ~90% storage cost reduction for old content |
| **Deduplication** | Perceptual hash before transcoding; skip if identical video exists | Prevents re-upload waste |
| **Client-side compression** | Mobile app compresses video before upload (target: 720p, 8Mbps) | Reduces upload size by 50-70% |

### Estimated Costs (at 10K active users)

Assumptions: 500 new videos/week, average 30s duration, average 20MB after client compression

| Component | Monthly Cost (est.) |
|-----------|-------------------|
| S3 storage (100GB growing) | $2-5 |
| MediaConvert transcoding | $25-50 |
| CloudFront CDN (500GB egress) | $40-60 |
| Total media costs | ~$70-115/month |

At 100K users these costs scale roughly 10x ($700-1,150/month).

---

## Database Configuration

### PostgreSQL (RDS)

```
Instance: db.t3.medium (MVP) → db.r6g.xlarge (scale)
Storage:  100GB gp3 (MVP) → auto-scaling
PostGIS:  CREATE EXTENSION IF NOT EXISTS postgis;
Backups:  Automated daily snapshots, 7-day retention
Read replica: Add when read load exceeds 60% CPU on primary
```

### Key PostgreSQL Settings
```
max_connections = 100           # Match connection pool size
shared_buffers = 256MB          # 25% of RAM for t3.medium
effective_cache_size = 768MB    # 75% of RAM
work_mem = 4MB                  # Per-operation sort memory
maintenance_work_mem = 64MB     # For VACUUM, CREATE INDEX
```

### Redis (ElastiCache)

```
Instance: cache.t3.small (MVP) → cache.r6g.large (scale)
Max memory: 1.5GB (MVP)
Eviction policy: volatile-lru (only evict keys with TTL set)
```

**Redis key namespaces:**
```
auth:blacklist:{token_hash}         # Blacklisted JWT tokens (TTL: token expiry)
session:{user_id}                   # Active session data
swipe_queue:{user_id}               # Pre-computed swipe queue (TTL: 30min)
swiped:{user_id}                    # Set of already-swiped user IDs
swipe_count:{user_id}:{date}        # Daily swipe counter (TTL: 24h)
superlike_count:{user_id}:{date}    # Daily superlike counter (TTL: 24h)
rate_limit:{endpoint}:{user_id}     # Sliding window rate limit counters
feed:foryou:{user_id}               # Cached feed (TTL: 15min)
post_likes:{post_id}                # Real-time like counter
spam_liker:{user_id}                # Flag for spam-like behavior (TTL: 7d)
online:{user_id}                    # Online presence (TTL: 5min, refreshed by heartbeat)
```

---

## Deployment

### MVP: Docker Compose (Local Dev)

```yaml
# docker-compose.yml (conceptual)
services:
  app:
    build: .
    ports: ["8000:8000"]
    depends_on: [postgres, redis, rabbitmq]

  celery-worker:
    build: .
    command: celery -A app.tasks.celery_app worker
    depends_on: [postgres, redis, rabbitmq]

  postgres:
    image: postgis/postgis:16-3.4
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  rabbitmq:
    image: rabbitmq:3-management
    ports: ["5672:5672", "15672:15672"]
```

### Production: AWS ECS Fargate

- **API**: 2+ ECS tasks behind ALB, auto-scaling on CPU/request count
- **Celery workers**: Separate ECS service, auto-scaling on queue depth
- **Database**: RDS PostgreSQL (Multi-AZ for production)
- **Redis**: ElastiCache (cluster mode for HA)
- **CI/CD**: GitHub Actions → ECR → ECS deploy

### Alternative: Simpler Stack for Budget MVP

If AWS is too expensive for initial launch:
- **Railway** or **Render**: FastAPI app + PostgreSQL + Redis in one platform
- **Cloudflare R2**: S3-compatible storage with free egress
- **Upstash**: Serverless Redis with pay-per-request
- Estimated cost: $25-50/month for low traffic

---

## Monitoring & Observability

| Component | Tool | Purpose |
|-----------|------|---------|
| Application logs | CloudWatch Logs / Datadog | Structured JSON logs |
| Metrics | CloudWatch Metrics / Prometheus | Request latency, error rates, queue depth |
| Uptime | AWS Health checks / UptimeRobot | API availability |
| Error tracking | Sentry | Exception capture with context |
| Database | RDS Performance Insights | Slow queries, connection pool |

### Key Alerts

| Alert | Condition | Action |
|-------|-----------|--------|
| API error rate > 5% | 5xx responses / total > 0.05 for 5 min | Page on-call |
| Celery queue backed up | > 100 pending tasks for > 10 min | Scale workers |
| Database CPU > 80% | Sustained for 10 min | Investigate queries, consider read replica |
| Moderation queue > 50 items | Unreviewed flagged content | Alert moderation team |
| Disk usage > 80% | On any volume | Expand storage |

---

## Security Infrastructure

| Layer | Implementation |
|-------|---------------|
| HTTPS | ALB terminates TLS (ACM certificate) |
| API authentication | JWT (RS256) with short-lived access tokens |
| Secrets management | AWS Secrets Manager (DB creds, API keys, JWT signing key) |
| Network | VPC with private subnets for DB/Redis, public subnet for ALB only |
| WAF | AWS WAF on ALB (rate limiting, SQL injection patterns, XSS) |
| Dependency scanning | Dependabot / Snyk for Python package vulnerabilities |
| Container scanning | ECR image scanning on push |
