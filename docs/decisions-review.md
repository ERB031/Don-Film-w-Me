# Decisions Review

Every key design decision extracted from the docs, organized by category. Mark each as APPROVED, REJECTED, or NEEDS DISCUSSION.

---

## Architecture

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| A1 | **Modular monolith** (not microservices) for MVP | Faster to build, deploy as single app, extract later if needed | |
| A2 | **FastAPI** (Python) as backend framework | Async support for real-time features, good for future ML | |
| A3 | **PostgreSQL 16 + PostGIS** for primary database | Relational data + geolocation queries in one DB | |
| A4 | **Redis** for caching, rate limiting, and swipe queues | Fast in-memory store, standard for this use case | |
| A5 | **Celery + RabbitMQ** for async tasks (transcoding, notifications) | Battle-tested Python task queue | |
| A6 | **AWS** as cloud provider (S3, ECS, RDS, CloudFront) | Comprehensive services, MediaConvert for video | |
| A7 | **Alternative budget stack** for MVP: Railway/Render + Cloudflare R2 (~$25-50/mo) | Avoid AWS complexity/cost early on | |

---

## User Model

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| U1 | **Single account, dual mode** (talent/client toggle) | Uber-style mode switch — simpler than separate accounts | |
| U2 | **1-hour cooldown** between mode switches, max 5/day | Prevents gaming (e.g., peeking at who liked you in other mode) | |
| U3 | **16 role types** (actor, model, DP, director, editor, gaffer, etc.) | Covers core film/photo industry roles | |
| U4 | **Users can have multiple roles** (actor AND model) with one marked primary | Reflects real industry — people wear many hats | |
| U5 | **Trust score (0-100)** starting at 50 for all users | Automated reputation system — gates features like cold booking requests | |
| U6 | **Trust score < 20 = auto-suspend** pending review | Hard floor to remove bad actors without manual monitoring | |

---

## Matching & Swiping

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| M1 | **Mutual like = match** (both swipe right) | Tinder-proven model — match unlocks messaging | |
| M2 | **100 swipes/day** (30 for new accounts < 7 days) | Prevents bots, forces intentional swiping | |
| M3 | **3 SuperLikes/day** with push notification to target | Premium signal of interest, higher match conversion | |
| M4 | **Swipe queue shows opposite mode only** (talent sees clients, clients see talent) | Core marketplace mechanic — supply meets demand | |
| M5 | **Scoring formula: distance (25) + recency (20) + role compatibility (20) + completeness (15) + trust (10) + popularity (10)** | Balanced ranking — no single factor dominates | |
| M6 | **Popularity capped at 10/100 points**, uses ratio not absolute numbers | Prevents winner-take-all where top profiles get all attention | |
| M7 | **Spam-liker detection**: >90% like ratio over 7 days → trust score reduction | Discourages mindless right-swiping | |
| M8 | **Swipe queue batch size: 20**, cached in Redis for 30 min | Balances freshness vs. performance | |
| M9 | **Queue diversification**: max 5 same role, max 7 same city per batch | Ensures variety in what users see | |
| M10 | **New user boost**: 15-point scoring bonus for first 48 hours | Solves cold start — new profiles actually get seen | |
| M11 | **Minimum profile to swipe**: completeness > 30, at least 1 approved photo + bio | No empty profiles in the queue | |
| M12 | **Swipe undo**: 3/day, within 5 seconds only | Prevents gaming while allowing genuine mistakes | |
| M13 | **Can only swipe on someone once** (unique constraint on swiper + swiped) | No re-swiping, keeps it clean | |

---

## Booking Requests

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| B1 | **Booking requests work with OR without a match** | Allows cold outreach for serious inquiries | |
| B2 | **Cold requests (no match) require 50+ char description** | Forces effort, filters spam | |
| B3 | **Cold request daily limits based on trust score** (0-5/day) | Higher trust = more access | |
| B4 | **Booking requests auto-expire after 7 days** | Prevents stale inbox clutter | |
| B5 | **Users can disable cold booking requests entirely** | Opt-out for those who only want matched interactions | |

---

## Content & Portfolio

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| C1 | **Full portfolio**: reels, photos, credits, resume, rate card, availability calendar | Comprehensive professional profile | |
| C2 | **Max 20 media items per portfolio** | Bounds storage costs, forces curation | |
| C3 | **60-second max video duration** for MVP | Controls transcoding costs (TikTok-length) | |
| C4 | **100MB max upload** (video), **20MB** (photo) | Client-side compression expected | |
| C5 | **Rate card visibility**: public / matches only / private (default: matches only) | Talent controls who sees their rates | |
| C6 | **Availability calendar visibility**: same 3 tiers (default: matches only) | Talent controls schedule exposure | |
| C7 | **EXIF metadata stripped** from all photo uploads | Prevents leaking GPS coordinates | |
| C8 | **Lazy 1080p transcoding**: only transcode to 1080p after 50+ views | 40-60% transcoding cost reduction | |
| C9 | **Cold storage after 180 days** unviewed → S3 Glacier | ~90% storage savings on old media | |

---

## Feed

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| F1 | **Two feeds**: "For You" (algorithm-ranked) + "Following" (chronological) | Algorithm for discovery, chronological for control | |
| F2 | **Feed sources**: 40% followed, 25% local, 20% trending, 15% role-relevant | Balanced mix of familiar + new content | |
| F3 | **Engagement score**: likes×1 + comments×2 + shares×3, log-scaled | Comments/shares weighted more than passive likes | |
| F4 | **Recency decay**: posts lose ranking power over ~48 hours | Keeps feed fresh, prevents stale viral posts | |
| F5 | **Video reels get +5 bonus** in feed scoring | Incentivizes richer content | |
| F6 | **Max 2 posts from same creator per feed batch** | Prevents any one person dominating the feed | |
| F7 | **Engagement farming detection**: velocity checks, like-source quality weighting | Auto-detects artificial engagement | |
| F8 | **Shadow reduction** (not shadow ban) for detected farmers | Content hidden from non-followers' For You, still visible to own followers | |
| F9 | **Max 15 hashtags per post**, known spam hashtags auto-flagged | Prevents hashtag stuffing | |

---

## Messaging

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| G1 | **Messaging requires a match** (no unsolicited DMs) | Core safety feature | |
| G2 | **Real-time via WebSocket** with REST fallback | Fast chat experience | |
| G3 | **60 messages/minute rate limit** per conversation | Prevents flooding | |
| G4 | **Unmatching deletes chat history** for both users | Clean break | |
| G5 | **Messages NOT end-to-end encrypted** | Allows moderation of reported conversations | |
| G6 | **URLs in messages trigger safety warning** to recipient | Anti-phishing | |
| G7 | **Identical messages to 3+ matches = auto-flag** | Catches copypasta spam | |

---

## Trust & Safety

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| T1 | **Phone verification required** to start swiping | Baseline identity check, blocks mass account creation | |
| T2 | **VoIP/virtual numbers blocked** at signup | Prevents cheap throwaway accounts | |
| T3 | **Optional ID verification** → badge + 10-point queue bonus | Incentivizes real identity without mandating it | |
| T4 | **3-tier moderation**: automated (Rekognition) → human review → community reports | Scalable content safety | |
| T5 | **Auto-moderation before any content goes public** | Nothing visible until at least auto-approved | |
| T6 | **3+ unique reporters on same target → auto-flag for priority review** | Community-driven safety signal | |
| T7 | **False reports reduce reporter's trust score** (-3 per dismissed report) | Discourages weaponized reporting | |
| T8 | **Blocking is invisible** to the blocked user (they just stop seeing you) | Prevents retaliation | |
| T9 | **Off-platform communication warning** in first 5 messages | Safety tip when phone/email/social detected in chat | |
| T10 | **Casting scam detection**: auto-warning when booking mentions upfront payment or private locations | Industry-specific safety | |
| T11 | **18+ age requirement** with dedicated "underage" report category (1-hour review SLA) | Legal compliance + minor safety | |
| T12 | **ID verification data deleted within 24 hours** of verification | Privacy — only boolean result retained | |

---

## Privacy

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| P1 | **Exact location never shared** — only city-level on profiles | Coordinates used server-side for distance only | |
| P2 | **Last active shown as "Active today/this week"**, not exact time | Prevents stalking | |
| P3 | **Soft delete with 30-day recovery**, then hard delete + anonymization | Balance between recovery and right-to-forget | |
| P4 | **GDPR/CCPA compliant**: data export, deletion, consent management | Legal requirement for any user-facing platform | |

---

## Monetization

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| $1 | **No monetization at MVP** — all features free | Build user base first | |
| $2 | **Freemium swipe limits**: 25/day free, unlimited with Pro | Familiar model, low friction | |
| $3 | **Boosts** ($4.99): 30-min visibility push in swipe queues | Quick impulse purchase | |
| $4 | **Feed Spotlight** ($2.99): feature a post for 24 hours | Content promotion | |
| $5 | **Pro subscription**: $9.99/mo (talent), $29.99/mo (client) | Asymmetric — talent pays less | |
| $6 | **Pro features**: unlimited swipes, 5 SuperLikes, see who liked you, analytics, read receipts | "See who liked you" is the killer conversion feature | |
| $7 | **Priority booking requests** ($7.99 each) | Cuts through noise for serious inquiries | |
| $8 | **Verified badge**: $9.99/mo | Pay for credibility | |
| $9 | **Booking transaction fee**: 5% client + 3% talent = 8% total | Main long-term revenue — marketplace cut | |
| $10 | **Escrow on bookings** — funds held until job confirmed complete | Key reason users won't go off-platform | |
| $11 | **Don Select B2B portal**: $199-$499/mo for businesses | Searchable talent directory for brands/agencies | |
| $12 | **Talent opt-in to Don Select** (toggle in settings) | Not exploitative — talent controls discoverability | |
| $13 | **Never paywall messaging** | Breaking the core loop kills the product | |
| $14 | **No display ads until 500K+ DAU** | Ads feel cheap in professional tool | |
| $15 | **Transaction fee capped at 8%** (never >10%) | Higher fees drive users off-platform | |

---

## Development Roadmap

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| R1 | **Phase 1 (Weeks 1-6)**: Auth + profiles + swipe + match + chat | Core loop must work first | |
| R2 | **Phase 2 (Weeks 7-12)**: Video reels + portfolio + feed + bookings | Content richness + public discovery | |
| R3 | **Phase 3 (Weeks 13-18)**: Verification + advanced filters + analytics + premium prep | Trust infrastructure + growth tools | |
| R4 | **Phase 4 (Weeks 19-30)**: Payments + reviews + casting calls + agency accounts + AI matching | Revenue + scale features | |
| R5 | **Target >60% test coverage** on services layer at MVP | Quality baseline | |
| R6 | **Mobile frontend tech TBD** (React Native or Flutter) | Decided later based on team/resources | |

---

## How to Use This Document

Review each decision and mark the Status column:
- **APPROVED** — Good to go, no changes needed
- **REJECTED** — Don't want this, explain why
- **MODIFY** — Concept is right but tweak the specifics (add a note)
- **DISCUSS** — Need more info or have questions

Decisions marked REJECTED or MODIFY will be reworked before implementation begins.
