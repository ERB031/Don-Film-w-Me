# Matching Algorithm

## Overview

The matching system has three stages:
1. **Queue Generation** — Build a ranked list of potential matches for each user
2. **Swipe Recording** — Track likes/passes efficiently
3. **Match Detection** — Detect mutual likes in real-time

---

## Stage 1: Swipe Queue Generation

When a user requests their swipe queue (`GET /discover/queue`), the system builds a personalized, ranked list of candidates.

### Step 1: Candidate Filtering (SQL)

Start with all users, then exclude ineligible ones:

```sql
SELECT u.id, u.display_name, u.avatar_url, u.city,
       ST_Distance(u.location, :user_location) AS distance_m,
       u.last_active_at, u.trust_score,
       p.completeness_score, p.headline
FROM users u
JOIN portfolios p ON p.user_id = u.id
WHERE
    -- Opposite mode: talent sees clients, clients see talent
    u.active_mode != :user_active_mode
    AND u.is_active = TRUE
    AND u.is_suspended = FALSE
    AND u.trust_score >= 20                          -- Minimum trust threshold

    -- Not already swiped
    AND u.id NOT IN (
        SELECT swiped_id FROM swipes WHERE swiper_id = :user_id
    )

    -- Not blocked (either direction)
    AND u.id NOT IN (
        SELECT user_b_id FROM matches WHERE user_a_id = :user_id AND status = 'blocked'
        UNION
        SELECT user_a_id FROM matches WHERE user_b_id = :user_id AND status = 'blocked'
    )

    -- Geo filter
    AND ST_DWithin(u.location, :user_location, :radius_meters)

    -- Role filter (optional, from query params)
    AND (:role_filter IS NULL OR EXISTS (
        SELECT 1 FROM user_roles ur WHERE ur.user_id = u.id AND ur.role = ANY(:role_filter)
    ))

    -- Has at least 1 approved media item (no empty profiles)
    AND EXISTS (
        SELECT 1 FROM media_items mi
        WHERE mi.user_id = u.id AND mi.moderation_status = 'approved'
    )
ORDER BY distance_m ASC  -- Initial sort by distance, re-ranked in step 2
LIMIT 200;               -- Fetch a larger pool for scoring
```

**Performance note**: The `NOT IN (SELECT swiped_id ...)` subquery is the most expensive part. For users with thousands of swipes, this is optimized by:
- Maintaining a Redis set `swiped:{user_id}` of all swiped user IDs
- Pre-filtering against this set before the SQL query
- Periodically rebuilding the set from the database

### Step 2: Scoring & Ranking

Each candidate from Step 1 gets a composite score:

```python
def score_candidate(candidate, viewer) -> float:
    score = 0.0

    # 1. Distance score (closer = better, max 25 points)
    distance_km = candidate.distance_m / 1000
    score += max(0, 25 - (distance_km * 0.5))

    # 2. Recency score (recently active = better, max 20 points)
    hours_since_active = hours_since(candidate.last_active_at)
    if hours_since_active < 1:
        score += 20
    elif hours_since_active < 24:
        score += 15
    elif hours_since_active < 72:
        score += 10
    elif hours_since_active < 168:  # 1 week
        score += 5
    # else: 0 (inactive users still shown but ranked low)

    # 3. Profile completeness (max 15 points)
    score += candidate.completeness_score * 0.15

    # 4. Role compatibility (max 20 points)
    if viewer.active_mode == 'client':
        # Client is seeking specific roles — exact match = bonus
        sought_roles = set(viewer.portfolio.seeking_roles or [])
        candidate_roles = set(candidate.roles)
        overlap = sought_roles & candidate_roles
        if overlap:
            score += 20
        elif candidate_roles:  # Has roles, just not what client seeks
            score += 5
    else:
        # Talent sees clients — score based on genre/style overlap
        viewer_genres = set(viewer.portfolio.genre_tags or [])
        candidate_genres = set(candidate.portfolio.genre_tags or [])
        genre_overlap = len(viewer_genres & candidate_genres)
        score += min(20, genre_overlap * 5)

    # 5. Trust/verification bonus (max 10 points)
    if candidate.verified:
        score += 10
    else:
        score += candidate.trust_score * 0.05  # 0-5 points based on trust

    # 6. Popularity signal — CAPPED to prevent winner-take-all (max 10 points)
    # Uses like-to-view ratio rather than raw like count
    if candidate.profile_views > 10:  # Need minimum views for signal
        like_ratio = candidate.likes_received / candidate.profile_views
        score += min(10, like_ratio * 20)

    return score
```

**Concern: Winner-Take-All / Popularity Bias**
- Popularity score is **capped at 10 points** out of 100 total
- Uses ratio (likes/views) not absolute numbers, so new users aren't disadvantaged
- "Boost" mechanics (paid or earned) are additive, not multiplicative

### Step 3: Diversification

After scoring, apply diversification rules before serving:

```python
def diversify_queue(scored_candidates, batch_size=20):
    queue = []
    seen_roles = Counter()
    seen_cities = Counter()

    for candidate in sorted(scored_candidates, key=lambda c: c.score, reverse=True):
        primary_role = candidate.primary_role

        # Don't show more than 5 of the same role consecutively
        if seen_roles[primary_role] >= 5 and len(queue) < batch_size:
            continue

        # Don't show more than 7 from the same city
        if seen_cities[candidate.city] >= 7 and len(queue) < batch_size:
            continue

        queue.append(candidate)
        seen_roles[primary_role] += 1
        seen_cities[candidate.city] += 1

        if len(queue) >= batch_size:
            break

    return queue
```

### Step 4: Caching

The generated queue is cached in Redis:

```
Key:    swipe_queue:{user_id}
Value:  Ordered list of candidate user IDs
TTL:    30 minutes (or until queue is exhausted)
```

- When the client requests cards, pop from this queue and hydrate with profile data
- When queue runs low (< 5 remaining), trigger background regeneration
- Queue is invalidated on: mode switch, location change, filter change

---

## Stage 2: Swipe Recording

When a user swipes:

```python
async def record_swipe(swiper_id, target_id, direction):
    # 1. Check daily limit
    today_count = await redis.get(f"swipe_count:{swiper_id}:{today()}")
    max_swipes = 30 if is_new_account(swiper_id) else 100
    if today_count >= max_swipes:
        raise RateLimitExceeded("Daily swipe limit reached")

    if direction == "superlike":
        superlike_count = await redis.get(f"superlike_count:{swiper_id}:{today()}")
        if superlike_count >= 3:
            raise RateLimitExceeded("Daily superlike limit reached")

    # 2. Record swipe in database
    swipe = Swipe(
        swiper_id=swiper_id,
        swiped_id=target_id,
        direction=direction,
        swiper_mode=current_user.active_mode
    )
    db.add(swipe)

    # 3. Add to swiped set in Redis (for queue filtering)
    await redis.sadd(f"swiped:{swiper_id}", target_id)

    # 4. Increment daily counter
    await redis.incr(f"swipe_count:{swiper_id}:{today()}")
    await redis.expire(f"swipe_count:{swiper_id}:{today()}", 86400)

    # 5. Check for match (if like or superlike)
    if direction in ("like", "superlike"):
        return await check_for_match(swiper_id, target_id, direction)

    return {"matched": False}
```

### Anti-Gaming: Spam-Like Detection

```python
async def detect_spam_liking(user_id):
    """
    If a user likes >90% of profiles they see, they're likely spam-swiping.
    Reduce their visibility in others' queues.
    """
    recent_swipes = await db.query(
        Swipe.direction
    ).filter(
        Swipe.swiper_id == user_id,
        Swipe.created_at > datetime.utcnow() - timedelta(days=7)
    ).all()

    if len(recent_swipes) < 20:
        return  # Not enough data

    like_ratio = sum(1 for s in recent_swipes if s.direction in ('like', 'superlike')) / len(recent_swipes)

    if like_ratio > 0.9:
        # Reduce trust score (affects their visibility in others' queues)
        await db.execute(
            update(User).where(User.id == user_id).values(
                trust_score=func.greatest(User.trust_score - 5, 10)
            )
        )
        # Their likes also carry less weight in the system
        await redis.set(f"spam_liker:{user_id}", "true", ex=86400 * 7)
```

---

## Stage 3: Match Detection

```python
async def check_for_match(swiper_id, target_id, direction):
    # Check if target has already liked swiper
    reverse_swipe = await db.query(Swipe).filter(
        Swipe.swiper_id == target_id,
        Swipe.swiped_id == swiper_id,
        Swipe.direction.in_(["like", "superlike"])
    ).first()

    if not reverse_swipe:
        # No mutual like yet
        if direction == "superlike":
            # Notify the target that they received a superlike
            await send_notification(
                target_id,
                type="superlike_received",
                title="Someone sent you a Super Like!",
                data={"screen": "discover"}
            )
        return {"matched": False}

    # MATCH FOUND — create match record
    user_a, user_b = sorted([swiper_id, target_id])  # Ensure a < b for uniqueness
    match = Match(user_a_id=user_a, user_b_id=user_b, status="active")
    db.add(match)
    await db.flush()

    # Notify both users
    await send_notification(
        swiper_id,
        type="new_match",
        title="It's a Match!",
        body=f"You and {target_user.display_name} liked each other",
        data={"screen": "match", "match_id": str(match.id)}
    )
    await send_notification(
        target_id,
        type="new_match",
        title="It's a Match!",
        body=f"You and {swiper_user.display_name} liked each other",
        data={"screen": "match", "match_id": str(match.id)}
    )

    return {"matched": True, "match_id": match.id}
```

---

## SuperLike Mechanics

SuperLikes are a premium signal of strong interest:

1. **Queue placement**: SuperLiked profiles appear with a badge in the target's queue, pushed toward the front (but not guaranteed #1 to prevent gaming)
2. **Notification**: Target gets a push notification immediately
3. **Higher match rate**: SuperLikes convert to matches ~3x more often than regular likes (based on industry data from similar apps)
4. **Limits**: 3 per day (free), additional purchasable in future premium tier

---

## Cold Start: New User Experience

New users face the "empty queue" problem. Mitigations:

1. **Curated seed profiles**: Manually verified, high-quality profiles marked as `is_featured` appear in every new user's first queue
2. **Broader geo radius**: New users' first few queues use a wider radius (100km vs default 50km)
3. **New user boost**: For the first 48 hours, new profiles get a 15-point scoring bonus in others' queues, ensuring they get seen
4. **Onboarding prompts**: Guide users to complete their profile before entering the swipe flow (completeness_score > 30 required)

---

## Scalability Considerations

### Current Design (MVP — up to ~50K users)
- Single PostgreSQL instance with read replica
- Redis for queue caching and rate limiting
- Queue regeneration is synchronous (fast enough for small user base)

### Scale-Up (50K–500K users)
- Move queue generation to Celery background workers
- Shard swipe history by user_id
- Add Elasticsearch for advanced profile search/filtering
- Pre-compute queues daily for active users, update incrementally

### Scale-Out (500K+ users)
- Dedicated matching service (separate deployment)
- Event-driven architecture (swipe events → Kafka → match checker)
- Geo-partitioned databases (US West, US East, Europe, etc.)
- ML-based scoring model trained on successful match/booking data
