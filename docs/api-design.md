# API Design

RESTful API built with FastAPI. All endpoints return JSON. Authentication via JWT Bearer tokens.

## Base URL

```
https://api.donfilmwme.com/v1
```

## Authentication

All endpoints except `/auth/*` require a valid JWT in the `Authorization: Bearer <token>` header.

### Auth Endpoints

```
POST /auth/register
  Body: {email, password, display_name, phone}
  Response: {user_id, access_token, refresh_token}
  Notes: Sends SMS OTP for phone verification

POST /auth/verify-phone
  Body: {phone, otp_code}
  Response: {verified: true}
  Notes: Required before user can start swiping

POST /auth/login
  Body: {email, password}
  Response: {access_token, refresh_token, user}

POST /auth/refresh
  Body: {refresh_token}
  Response: {access_token, refresh_token}

POST /auth/logout
  Body: {refresh_token}
  Response: 204 No Content
  Notes: Blacklists refresh token in Redis

POST /auth/oauth/{provider}
  Body: {id_token}     (provider: google | apple)
  Response: {access_token, refresh_token, user, is_new_user}

POST /auth/forgot-password
  Body: {email}
  Response: 202 Accepted
  Notes: Sends password reset email with 15-min expiry token

POST /auth/reset-password
  Body: {reset_token, new_password}
  Response: 200 OK
```

**Security measures:**
- Passwords hashed with bcrypt (work factor 12)
- Access tokens expire in 15 minutes
- Refresh tokens expire in 30 days, single-use (rotated on each refresh)
- Failed login rate limiting: 5 attempts per 15 minutes per email, then 30-min lockout
- OAuth tokens validated server-side against provider's public keys

---

## User Profile

```
GET /users/me
  Response: {user object with portfolio, roles, stats}

PATCH /users/me
  Body: {display_name?, bio?, city?, location?}
  Response: {updated user}

PATCH /users/me/mode
  Body: {active_mode: "talent" | "client"}
  Response: {user with new mode}
  Errors:
    429 - Mode switch cooldown (1 hour minimum between switches)
    422 - Incomplete profile for requested mode

GET /users/{id}/profile
  Response: {public profile view — respects privacy settings}
  Notes: Rate card hidden unless visibility allows it.
         Returns 404 if user has blocked the viewer.

DELETE /users/me
  Response: 202 Accepted
  Notes: Soft delete. Data retained 30 days for recovery, then hard deleted.
         Triggers: remove from all swipe queues, unmatch all matches,
         cancel pending bookings, anonymize feed posts.
```

---

## Portfolio & Media

```
GET /portfolio/me
  Response: {portfolio with media_items, credits, rate_card}

PUT /portfolio/me
  Body: {headline?, resume_text?, credits?, rate_min?, rate_max?,
         rate_unit?, rate_visibility?, seeking_roles?, genre_tags?, style_tags?}
  Response: {updated portfolio}

POST /portfolio/media
  Body: multipart/form-data {file, media_type, title?, description?}
  Response: {media_item with processing_status: "pending"}
  Limits:
    - Max file size: 100MB (video), 20MB (photo)
    - Max duration: 60 seconds (video)
    - Accepted formats: mp4, mov, jpg, jpeg, png, webp
    - Max 20 media items per portfolio
  Notes: Returns immediately. Transcoding happens async via Celery.
         Client polls GET /portfolio/media/{id} for processing_status.

GET /portfolio/media/{id}
  Response: {media_item with current processing/moderation status}

DELETE /portfolio/media/{id}
  Response: 204 No Content
  Notes: Removes from S3 after 24h grace period (in case of accidental delete)

PATCH /portfolio/media/reorder
  Body: {media_ids: [ordered list of UUIDs]}
  Response: 200 OK
```

**Concern: Malicious File Uploads**
- MIME type validated server-side (not just extension)
- Files scanned with ClamAV before processing
- Video transcoded through isolated FFmpeg workers (sandboxed, no network access)
- Metadata stripped from all uploads (EXIF data may contain GPS coordinates)

---

## Discovery (Swipe)

```
GET /discover/queue?limit=20
  Query params:
    roles[]     - Filter by talent roles (actor, model, dp, etc.)
    radius_km   - Max distance (default: 50, max: 500)
    min_rate    - Minimum rate filter
    max_rate    - Maximum rate filter
    genre[]     - Genre preference filter
    available_on - Date to check availability
  Response: {
    cards: [{
      user_id, display_name, avatar_url, city, distance_km,
      roles: [], headline, primary_media: {url, thumbnail, type},
      portfolio_preview: [top 3 media thumbnails],
      compatibility_score: 0.0-1.0
    }],
    remaining: int,
    refresh_at: timestamp    // When to request next batch
  }
  Notes:
    - Returns users in the OPPOSITE mode (talent sees clients, clients see talent)
    - Excludes: already-swiped, blocked, suspended, unverified users
    - Pre-fetches next batch in background when remaining < 5

POST /discover/swipe
  Body: {target_id, direction: "like" | "pass" | "superlike"}
  Response: {
    matched: boolean,
    match_id?: UUID,          // If matched
    remaining_swipes: int,    // Daily swipes remaining
    remaining_superlikes: int
  }
  Rate limits:
    - 100 swipes per 24h (likes + passes + superlikes)
    - 3 superlikes per 24h
    - New accounts (< 7 days): 30 swipes per 24h
  Errors:
    429 - Daily swipe limit reached
    409 - Already swiped on this user

POST /discover/undo
  Response: {undone: true}
  Notes: Undo last swipe (within 5 seconds only, 3 undos per day)
```

---

## Matches

```
GET /matches?page=1&per_page=20
  Response: {
    matches: [{
      match_id, user: {id, display_name, avatar_url, headline, roles},
      matched_at, last_message?: {content, created_at},
      unread_count: int
    }],
    total: int
  }

DELETE /matches/{id}
  Response: 204 No Content
  Notes: Unmatches. Deletes chat history. Both users notified.
         Cannot be undone.

POST /matches/{id}/block
  Response: 204 No Content
  Notes: Unmatches + blocks. Blocked user cannot see blocker's profile,
         appear in their queue, or send booking requests.
```

---

## Booking Requests

```
POST /bookings
  Body: {
    target_id,
    match_id?,              // Optional — null for cold requests
    project_title,
    project_description,    // Min 50 characters for cold requests
    project_type?,
    proposed_dates?,        // [{start: date, end: date}]
    location_description?,
    proposed_rate?,
    rate_unit?
  }
  Response: {booking_request}
  Rate limits (cold requests only — no match_id):
    - 5 per day (trust_score >= 70)
    - 3 per day (trust_score 40-69)
    - 1 per day (trust_score < 40)
    - 0 per day (trust_score < 20 — must verify identity first)
  Errors:
    429 - Daily cold request limit reached
    403 - Target has blocked you or disabled cold requests
    422 - Description too short for cold request

GET /bookings/incoming?status=pending
  Response: {booking_requests[]}

GET /bookings/outgoing?status=pending
  Response: {booking_requests[]}

PATCH /bookings/{id}
  Body: {status: "accepted" | "declined", response_message?}
  Response: {updated booking_request}
  Notes: Only the target can accept/decline.
         Accepting sends notification to requester.

DELETE /bookings/{id}
  Response: 204 No Content
  Notes: Requester can withdraw pending requests only.
```

---

## Feed

```
GET /feed?cursor=<timestamp>&limit=20
  Response: {
    posts: [{
      id, user: {id, display_name, avatar_url, verified},
      media: {url, thumbnail_url, type, duration},
      caption, hashtags,
      like_count, comment_count,
      liked_by_me: boolean,
      created_at
    }],
    next_cursor: timestamp
  }
  Notes: Cursor-based pagination for infinite scroll.
         Algorithm-ranked (see feed-algorithm.md).

GET /feed/following?cursor=<timestamp>&limit=20
  Response: Same as above but only from followed users, chronological order

POST /feed/posts
  Body: {media_item_id?, caption?, hashtags?[]}
  Response: {feed_post}
  Notes: Media must already be uploaded via /portfolio/media
         and have moderation_status = 'approved'

DELETE /feed/posts/{id}
  Response: 204 No Content

POST /feed/posts/{id}/like
  Response: 200 {liked: true, like_count: int}

DELETE /feed/posts/{id}/like
  Response: 200 {liked: false, like_count: int}

GET /feed/posts/{id}/comments?cursor=&limit=20
  Response: {comments[]}

POST /feed/posts/{id}/comments
  Body: {body, parent_id?}
  Response: {comment}
  Rate limit: 30 comments per hour

POST /users/{id}/follow
  Response: 200 {following: true}

DELETE /users/{id}/follow
  Response: 200 {following: false}
```

---

## Messaging

```
GET /messages/{match_id}?cursor=<timestamp>&limit=50
  Response: {messages[], next_cursor}

POST /messages/{match_id}
  Body: {content, message_type?: "text" | "image" | "booking_link"}
  Response: {message}
  Notes: Also broadcast via WebSocket to recipient
  Rate limit: 60 messages per minute (prevents flooding)
```

### WebSocket

```
WS /ws/chat
  Auth: JWT passed as query param or first message

  Client → Server events:
    {type: "send_message", match_id, content}
    {type: "typing", match_id}
    {type: "read", match_id, message_id}

  Server → Client events:
    {type: "new_message", match_id, message}
    {type: "typing", match_id, user_id}
    {type: "read_receipt", match_id, message_id, read_at}
    {type: "new_match", match}
    {type: "match_unmatched", match_id}
```

---

## Reporting

```
POST /reports
  Body: {
    reported_user_id?,
    reported_post_id?,
    reported_message_id?,
    reason: enum,
    description?,
    evidence_urls?[]
  }
  Response: {report_id, status: "pending"}
  Notes: At least one of reported_user_id, reported_post_id,
         or reported_message_id must be provided.
         Duplicate reports (same reporter + same target) return existing report.
```

---

## Availability

```
GET /availability/me?month=2026-04
  Response: {slots: [{date, status, notes}]}

PUT /availability/me
  Body: {slots: [{date, status, visibility?, notes?}]}
  Response: {updated slots}

GET /users/{id}/availability?month=2026-04
  Response: {slots[]}  // Filtered by visibility settings
  Errors:
    403 - Availability is private and you don't have access
```

---

## Common Error Format

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Daily swipe limit reached. Resets at 2026-03-23T00:00:00Z",
    "details": {
      "limit": 100,
      "used": 100,
      "resets_at": "2026-03-23T00:00:00Z"
    }
  }
}
```

## Rate Limiting Strategy

All rate limits use a sliding window algorithm backed by Redis.

| Endpoint | Limit | Window | Notes |
|----------|-------|--------|-------|
| POST /auth/login | 5 | 15 min | Per email |
| POST /discover/swipe | 100 | 24h | Per user |
| POST /discover/swipe (superlike) | 3 | 24h | Per user |
| POST /bookings (cold) | 1-5 | 24h | Based on trust_score |
| POST /feed/posts/{id}/comments | 30 | 1h | Per user |
| POST /messages/{match_id} | 60 | 1 min | Per user per match |
| POST /reports | 10 | 24h | Per reporter |
| PATCH /users/me/mode | 5 | 24h | Per user |
| POST /portfolio/media | 10 | 1h | Per user |
