# Don Film w/ Me

A mobile-first talent matching platform for the film and photography industry. Think Tinder meets TikTok meets a professional casting service.

## What It Does

- **Swipe Discovery** — Browse talent and clients through a card-based swiping interface. Filter by role, location, genre, and availability.
- **Public Feed** — A vertical scrollable feed (TikTok/IG-style) where creatives post reels and photos for organic discovery.
- **Matching** — Mutual likes create a match, unlocking direct messaging. Both sides swipe independently.
- **Booking Requests** — Send project proposals to talent, even without a match (with anti-spam safeguards).
- **Full Portfolios** — Video reels, photo galleries, credits/resume, rate cards, and availability calendars.
- **Hybrid Mode** — One account, two modes. Toggle between "Looking for Work" (talent) and "Hiring" (client) like Uber's driver/rider switch.

## Target Users

| Role | Examples |
|------|----------|
| Talent | Actors, models, dancers, voice artists |
| Crew | Directors of photography, editors, gaffers, sound engineers, stylists, makeup artists |
| Creatives | Photographers, directors, producers |
| Clients | Production companies, brands, agencies, independent filmmakers |

## Tech Stack

- **Backend**: Python / FastAPI (async)
- **Database**: PostgreSQL 16 + PostGIS (geolocation)
- **Cache & Queues**: Redis + Celery + RabbitMQ
- **Storage**: S3-compatible object storage + CDN
- **Mobile**: TBD (React Native / Flutter)

## Documentation

All system design documents live in [`docs/`](./docs/):

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture-overview.md) | Services, data flow, mode system |
| [Database Schema](docs/database-schema.md) | All tables, relationships, indexes |
| [API Design](docs/api-design.md) | REST endpoints, WebSocket contracts |
| [Matching Algorithm](docs/matching-algorithm.md) | Swipe queue, scoring, match detection |
| [Feed Algorithm](docs/feed-algorithm.md) | Content ranking, cold start, diversification |
| [Trust & Safety](docs/trust-safety.md) | Anti-spam, moderation, privacy, abuse prevention |
| [Infrastructure](docs/infrastructure.md) | Deployment, storage, transcoding, scaling |
| [Development Roadmap](docs/development-roadmap.md) | Phased plan: MVP → v1 → v2 |

## Status

**Phase: Design** — Architecture and system design in progress. No code yet.
