# Development Roadmap

## Phase 1: MVP (Weeks 1-6)

**Goal**: Core swipe-to-match flow working end-to-end. Users can sign up, create a profile, swipe, match, and message.

### Week 1-2: Foundation
- [ ] Project setup (FastAPI, Docker Compose, PostgreSQL + PostGIS, Redis)
- [ ] Alembic migrations for core tables (users, user_roles, portfolios, media_items)
- [ ] User registration + login (email/password + JWT)
- [ ] Phone verification via SMS OTP (Twilio)
- [ ] Basic profile CRUD (display name, bio, avatar, city, roles)
- [ ] Mode toggle endpoint (talent ↔ client)

### Week 3-4: Swipe & Match
- [ ] Photo upload to S3 with basic auto-moderation (NSFW check)
- [ ] Swipe queue generation (geo + role filtering, basic scoring)
- [ ] Swipe recording with daily rate limits
- [ ] Match detection (mutual like → match created)
- [ ] Swipe queue caching in Redis
- [ ] SuperLike (limited to 3/day)

### Week 5-6: Messaging & Polish
- [ ] Match list endpoint
- [ ] WebSocket chat (text messages only)
- [ ] Push notifications (FCM) for: new match, new message
- [ ] Block and unmatch functionality
- [ ] Basic reporting (submit report → stored in DB)
- [ ] Profile completeness scoring
- [ ] Spam-liker detection (>90% like ratio)

### MVP Exit Criteria
- [ ] User can: register → verify phone → create profile → set mode → upload photos → swipe → match → chat
- [ ] Rate limiting on all endpoints
- [ ] Auto-moderation catches NSFW uploads
- [ ] Trust score initialized for all users
- [ ] Basic test coverage (>60% on services)

---

## Phase 2: Content & Feed (Weeks 7-12)

**Goal**: Full portfolio system and public feed. Video reel support. Booking requests.

### Week 7-8: Video & Portfolio
- [ ] Video reel upload (60s max, 100MB max)
- [ ] Transcoding pipeline (Celery + FFmpeg/MediaConvert → 720p + thumbnail)
- [ ] Full portfolio: credits list, resume text, headline
- [ ] Rate card (min/max rates, visibility settings)
- [ ] Availability calendar (CRUD for date slots)
- [ ] Media reordering in portfolio

### Week 9-10: Public Feed
- [ ] Feed post creation (link to approved media + caption)
- [ ] "For You" feed with ranking algorithm
- [ ] "Following" feed (chronological)
- [ ] Like, comment, share on feed posts
- [ ] Follow/unfollow users
- [ ] Hashtag support with GIN index search
- [ ] Feed diversification (no more than 2 posts from same creator per batch)
- [ ] Cold start feed for new users (featured + local trending)

### Week 11-12: Bookings & Moderation
- [ ] Booking request system (create, accept, decline, withdraw)
- [ ] Cold booking requests (without match) with trust-gated rate limits
- [ ] Auto-expiry of pending requests (7 days)
- [ ] Moderation queue for flagged content (admin endpoint)
- [ ] Community reporting with auto-escalation (3+ reporters → priority)
- [ ] Content appeal process
- [ ] Admin action audit log

### Phase 2 Exit Criteria
- [ ] Video reels play in swipe cards and feed
- [ ] Feed algorithm produces relevant, diverse content
- [ ] Booking requests work for both matched and unmatched users
- [ ] Moderation pipeline: auto-flag → queue → review → resolve
- [ ] Cold storage migration for old media (>180 days unviewed)

---

## Phase 3: Trust & Growth (Weeks 13-18)

**Goal**: Identity verification, advanced matching, analytics, and premium features.

### Week 13-14: Verification & Trust
- [ ] Identity verification flow (ID upload → selfie comparison → badge)
- [ ] Trust score system fully implemented (all gain/loss rules)
- [ ] Professional verification (IMDB/LinkedIn link validation)
- [ ] Engagement farming detection (velocity checks, like source quality)
- [ ] Shadow reduction for detected spam accounts
- [ ] Mode-switching abuse detection and cooldowns

### Week 15-16: Discovery Improvements
- [ ] Advanced swipe filters (genre, style, rate range, availability date)
- [ ] Profile search (by name, role, city) — Elasticsearch
- [ ] "Explore" tab: browse by category/role/trending
- [ ] New user boost (48h visibility increase)
- [ ] Swipe undo feature (3/day, 5-second window)
- [ ] Improved scoring: style/genre compatibility

### Week 17-18: Analytics & Premium
- [ ] Creator analytics dashboard: profile views, swipe-right rate, match rate, booking conversion
- [ ] Client analytics: response rate, average time to accept
- [ ] Push notification preferences (granular opt-in/out)
- [ ] Email digest (weekly summary of matches, bookings, feed highlights)
- [ ] Premium tier groundwork: extra swipes, unlimited superlikes, see who liked you, read receipts

### Phase 3 Exit Criteria
- [ ] Verified users have visible badge, higher queue ranking
- [ ] Trust score actively gates cold request limits
- [ ] Search and explore provide alternative discovery paths
- [ ] Analytics give users actionable insights on their performance

---

## Phase 4: Scale & Monetization (Weeks 19-30)

**Goal**: Revenue generation, team features, AI-powered matching.

### Revenue Streams (to evaluate)
- [ ] **Premium subscription**: Extra swipes, superlikes, see-who-liked, priority queue placement, read receipts, profile boost
- [ ] **Booking transaction fee**: Small % on completed bookings (if in-app payments added)
- [ ] **Promoted profiles**: Pay to appear higher in swipe queues for a set period
- [ ] **Featured feed posts**: Pay to promote a post in the "For You" feed

### Features
- [ ] In-app payments for bookings (Stripe Connect)
- [ ] Review/rating system post-booking completion
- [ ] Project/casting call posting (client creates a "casting call" visible to matching talent)
- [ ] Crew assembly tool: build a full team (DP + gaffer + sound + stylist) for a project
- [ ] Agency accounts: manage multiple talent profiles under one organization
- [ ] AI-powered matching: analyze reel style, shooting aesthetic, past collaboration success
- [ ] Direct calendar integration (Google Calendar / Apple Calendar sync)

### Infrastructure Scale
- [ ] Microservice extraction if single service becomes bottleneck
- [ ] Geo-sharded databases for international expansion
- [ ] ML ranking model for feed and swipe queue (LightGBM trained on engagement data)
- [ ] Real-time analytics pipeline (event streaming)

---

## Technical Debt & Maintenance (Ongoing)

| Cadence | Task |
|---------|------|
| Weekly | Dependency updates (Dependabot PRs) |
| Weekly | Review moderation queue accuracy (false positive rate) |
| Bi-weekly | Database query performance review (slow query log) |
| Monthly | Security audit (container scans, credential rotation) |
| Monthly | Cost review (AWS billing, identify optimization opportunities) |
| Quarterly | Trust & Safety transparency report |
| Quarterly | Load testing with realistic user simulation |

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Cold start — not enough users to make swiping useful | High | High | Seed with curated profiles, target specific cities for launch, partner with talent agencies |
| Video costs spiral with growth | Medium | Medium | Aggressive caching, lazy transcoding, client-side compression, cold storage |
| Content moderation fails to catch harmful content | High | Medium | Multi-tier moderation, community reporting, strict SLAs on review time |
| Users prefer existing platforms (Instagram, Casting Networks) | High | High | Differentiate with matching mechanic + booking workflow — features IG doesn't have |
| Mode-switching creates confusing UX | Medium | Medium | Clear UI indicators of current mode, confirmation on switch, separate notification channels |
| Legal/compliance issues (GDPR, age verification, labor laws) | High | Low | Consult legal counsel before launch, implement data rights features early |
