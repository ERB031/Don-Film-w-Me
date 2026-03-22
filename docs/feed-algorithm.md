# Feed Algorithm

## Overview

The public feed serves two purposes:
1. **Organic discovery** — Users find talent/clients they wouldn't encounter through swiping alone
2. **Content showcase** — Creatives post work samples, behind-the-scenes, and finished projects

Two feed views exist:
- **"For You" feed** — Algorithm-ranked, mixes followed and unfollowed creators
- **"Following" feed** — Chronological, only from followed users

---

## "For You" Feed: Ranking Algorithm

### Step 1: Candidate Pool Assembly

For each feed request, pull candidates from multiple sources:

```python
def assemble_candidate_pool(viewer, limit=200):
    candidates = []

    # Source 1: Posts from followed users (40% of pool)
    followed_posts = db.query(FeedPost).join(Follow,
        Follow.followed_id == FeedPost.user_id
    ).filter(
        Follow.follower_id == viewer.id,
        FeedPost.moderation_status == 'approved',
        FeedPost.created_at > datetime.utcnow() - timedelta(days=7)
    ).order_by(FeedPost.created_at.desc()).limit(80).all()
    candidates.extend(followed_posts)

    # Source 2: Local posts — same city or within 50km (25% of pool)
    local_posts = db.query(FeedPost).join(User).filter(
        FeedPost.moderation_status == 'approved',
        FeedPost.created_at > datetime.utcnow() - timedelta(days=3),
        ST_DWithin(User.location, viewer.location, 50000),  # 50km
        FeedPost.user_id != viewer.id
    ).order_by(FeedPost.like_count.desc()).limit(50).all()
    candidates.extend(local_posts)

    # Source 3: Trending posts — high engagement, any location (20% of pool)
    trending_posts = db.query(FeedPost).filter(
        FeedPost.moderation_status == 'approved',
        FeedPost.created_at > datetime.utcnow() - timedelta(days=2),
        FeedPost.like_count >= 10  # Minimum engagement threshold
    ).order_by(FeedPost.like_count.desc()).limit(40).all()
    candidates.extend(trending_posts)

    # Source 4: Role-relevant posts (15% of pool)
    if viewer.active_mode == 'client' and viewer.portfolio.seeking_roles:
        role_posts = db.query(FeedPost).join(User).join(UserRole).filter(
            FeedPost.moderation_status == 'approved',
            FeedPost.created_at > datetime.utcnow() - timedelta(days=5),
            UserRole.role.in_(viewer.portfolio.seeking_roles)
        ).order_by(FeedPost.created_at.desc()).limit(30).all()
        candidates.extend(role_posts)

    # Deduplicate
    seen_ids = set()
    unique_candidates = []
    for post in candidates:
        if post.id not in seen_ids:
            seen_ids.add(post.id)
            unique_candidates.append(post)

    return unique_candidates
```

### Step 2: Scoring

Each candidate post gets a composite score:

```python
def score_post(post, viewer) -> float:
    score = 0.0

    # 1. Engagement score (max 30 points)
    engagement = (
        post.like_count * 1.0 +
        post.comment_count * 2.0 +
        post.share_count * 3.0
    )
    # Log scale to prevent viral posts from dominating forever
    score += min(30, math.log1p(engagement) * 5)

    # 2. Recency decay (multiplier: 1.0 for fresh, decays over time)
    hours_old = hours_since(post.created_at)
    recency_multiplier = 1.0 / (1.0 + hours_old * 0.05)
    # Posts older than 48 hours get significant decay

    # 3. Follow bonus (max 15 points)
    if is_following(viewer.id, post.user_id):
        score += 15

    # 4. Role relevance (max 10 points)
    creator_roles = get_roles(post.user_id)
    if viewer.active_mode == 'client':
        sought = set(viewer.portfolio.seeking_roles or [])
        if sought & set(creator_roles):
            score += 10
    else:
        # Talent mode: boost posts from potential clients
        if get_user_mode(post.user_id) == 'client':
            score += 5

    # 5. Locality bonus (max 10 points)
    distance = get_distance(viewer.location, get_user_location(post.user_id))
    if distance < 10:       # Same area
        score += 10
    elif distance < 50:     # Same metro
        score += 5
    elif distance < 100:
        score += 2

    # 6. Content type bonus
    if post.media_item and post.media_item.media_type == 'video_reel':
        score += 5  # Video reels get slight priority (higher engagement potential)

    # 7. Creator quality signal (max 5 points)
    creator = get_user(post.user_id)
    if creator.verified:
        score += 3
    score += creator.trust_score * 0.02  # 0-2 points

    # Apply recency multiplier to final score
    return score * recency_multiplier
```

### Step 3: Diversification

Prevent feed fatigue by enforcing variety:

```python
def diversify_feed(scored_posts, batch_size=20):
    feed = []
    creator_counts = Counter()
    role_counts = Counter()
    last_media_type = None
    consecutive_same_type = 0

    for post in sorted(scored_posts, key=lambda p: p.score, reverse=True):
        creator_id = post.user_id
        creator_role = get_primary_role(creator_id)

        # Rule 1: Max 2 posts from same creator per batch
        if creator_counts[creator_id] >= 2:
            continue

        # Rule 2: Max 5 posts of same role type per batch
        if role_counts[creator_role] >= 5:
            continue

        # Rule 3: Don't show 4+ videos or 4+ photos in a row
        media_type = post.media_item.media_type if post.media_item else 'text'
        if media_type == last_media_type:
            consecutive_same_type += 1
            if consecutive_same_type >= 3:
                continue
        else:
            consecutive_same_type = 0

        feed.append(post)
        creator_counts[creator_id] += 1
        role_counts[creator_role] += 1
        last_media_type = media_type

        if len(feed) >= batch_size:
            break

    return feed
```

---

## "Following" Feed

Simple reverse-chronological feed from followed users only:

```sql
SELECT fp.*, u.display_name, u.avatar_url
FROM feed_posts fp
JOIN follows f ON f.followed_id = fp.user_id
JOIN users u ON u.id = fp.user_id
WHERE f.follower_id = :viewer_id
  AND fp.moderation_status = 'approved'
  AND fp.created_at < :cursor
ORDER BY fp.created_at DESC
LIMIT 20;
```

No ranking, no algorithm — pure chronological. This gives users a predictable, transparent alternative.

---

## Cold Start: New Users

New users with no follows and no interaction history get a special feed:

```python
def cold_start_feed(viewer, limit=20):
    posts = []

    # 1. Featured/curated posts (hand-picked by team)
    featured = db.query(FeedPost).filter(
        FeedPost.is_pinned == True,
        FeedPost.moderation_status == 'approved'
    ).order_by(func.random()).limit(5).all()
    posts.extend(featured)

    # 2. Trending in their city
    local_trending = db.query(FeedPost).join(User).filter(
        User.city == viewer.city,
        FeedPost.moderation_status == 'approved',
        FeedPost.created_at > datetime.utcnow() - timedelta(days=3),
        FeedPost.like_count >= 5
    ).order_by(FeedPost.like_count.desc()).limit(10).all()
    posts.extend(local_trending)

    # 3. Top posts by role type matching their interests
    if viewer.portfolio and viewer.portfolio.seeking_roles:
        role_posts = db.query(FeedPost).join(User).join(UserRole).filter(
            UserRole.role.in_(viewer.portfolio.seeking_roles),
            FeedPost.moderation_status == 'approved',
            FeedPost.created_at > datetime.utcnow() - timedelta(days=7)
        ).order_by(FeedPost.like_count.desc()).limit(10).all()
        posts.extend(role_posts)

    # 4. Globally trending (fallback)
    if len(posts) < limit:
        global_trending = db.query(FeedPost).filter(
            FeedPost.moderation_status == 'approved',
            FeedPost.created_at > datetime.utcnow() - timedelta(days=2)
        ).order_by(FeedPost.like_count.desc()).limit(limit - len(posts)).all()
        posts.extend(global_trending)

    return deduplicate(posts)[:limit]
```

---

## Concern: Feed Manipulation / Engagement Farming

**Risk**: Users could artificially boost their posts through like-farms, follow-for-follow schemes, or bot engagement.

**Mitigations**:

1. **Engagement velocity checks**: If a post gets 50+ likes in < 1 minute from creation, auto-flag for review. Organic growth follows a curve, not a spike.

2. **Like source quality**: Likes from accounts with trust_score < 30 count at 0.25x weight in the engagement score. Likes from verified accounts count at 1.5x.

3. **Follow reciprocity detection**: If two accounts follow each other within 30 seconds, neither follow counts toward feed ranking (though the follow relationship still exists).

4. **Hashtag spam**: Posts with > 15 hashtags get a scoring penalty. Known spam hashtags (maintained list) trigger auto-flag.

5. **Duplicate content detection**: Perceptual hashing (pHash) on uploaded media. If the same image/video is posted multiple times by different accounts, flag the newer copies.

6. **Shadow reduction** (not shadow banning): Accounts detected as engagement-farming don't get banned outright — their content just stops appearing in the "For You" feed of non-followers. They can still post and appear in followers' feeds. This avoids false-positive punishment while limiting damage.

---

## Caching Strategy

```
# Pre-computed feed for active users (refreshed every 15 minutes)
Key:    feed:foryou:{user_id}
Value:  Ordered list of post IDs with scores
TTL:    15 minutes

# Post data cache (shared across all users)
Key:    post:{post_id}
Value:  Serialized post data (JSON)
TTL:    5 minutes

# Engagement counters (updated in real-time)
Key:    post_likes:{post_id}
Value:  Integer count
TTL:    None (persistent, synced to DB every 5 minutes)
```

---

## Future: ML-Based Ranking (v2+)

Once sufficient interaction data is collected (est. 6+ months post-launch):

- Train a simple ranking model on features: {viewer_roles, creator_roles, distance, post_age, engagement_rate, has_video, viewer_liked_similar}
- Target variable: did the viewer engage (like, comment, follow, or swipe on creator)?
- Model: Gradient boosted trees (LightGBM) — fast inference, interpretable
- Retrain weekly on latest 30 days of data
