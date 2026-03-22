# Database Schema

PostgreSQL 16 with PostGIS extension for geolocation queries.

## Entity Relationship Diagram (Simplified)

```
users ─┬─< user_roles
       ├─< portfolios ──< media_items
       ├─< availability_slots
       ├─< swipes (as swiper)
       ├─< swipes (as swiped)
       ├─< matches (as user_a or user_b)
       ├─< booking_requests (as requester or target)
       ├─< follows (as follower or followed)
       ├─< feed_posts ──< feed_likes
       │                ├─< feed_comments
       ├─< messages (as sender)
       ├─< notifications
       └─< reports (as reporter or reported)
```

## Tables

### users

Primary user account. Shared across both modes.

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    phone           VARCHAR(20) UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    avatar_url      TEXT,

    -- Mode system
    active_mode     VARCHAR(10) NOT NULL DEFAULT 'talent'
                    CHECK (active_mode IN ('talent', 'client')),
    last_mode_switch TIMESTAMPTZ,

    -- Profile
    bio             TEXT,
    location        GEOGRAPHY(POINT, 4326),  -- PostGIS point (lng, lat)
    city            VARCHAR(100),
    state           VARCHAR(100),
    country         VARCHAR(3),               -- ISO 3166-1 alpha-3

    -- Verification & trust
    verified        BOOLEAN DEFAULT FALSE,
    verification_method VARCHAR(20),          -- 'phone', 'id', 'portfolio_review'
    trust_score     SMALLINT DEFAULT 50       -- 0-100, affects visibility
                    CHECK (trust_score >= 0 AND trust_score <= 100),

    -- Status
    is_active       BOOLEAN DEFAULT TRUE,
    is_suspended    BOOLEAN DEFAULT FALSE,
    suspension_reason TEXT,
    last_active_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_active_mode ON users (active_mode) WHERE is_active = TRUE;
CREATE INDEX idx_users_location ON users USING GIST (location);
CREATE INDEX idx_users_last_active ON users (last_active_at DESC);
CREATE INDEX idx_users_city ON users (city, active_mode);
```

### user_roles

A user can have multiple professional roles (e.g., actor AND model).

```sql
CREATE TYPE role_type AS ENUM (
    'actor', 'model', 'dancer', 'voice_artist',
    'photographer', 'director', 'producer',
    'dp', 'editor', 'gaffer', 'sound_engineer',
    'stylist', 'makeup_artist', 'wardrobe',
    'production_assistant', 'other'
);

CREATE TABLE user_roles (
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    role        role_type NOT NULL,
    experience_years SMALLINT,
    is_primary  BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (user_id, role)
);

CREATE INDEX idx_user_roles_role ON user_roles (role);
```

### portfolios

Professional portfolio linked to user account. One per user.

```sql
CREATE TABLE portfolios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    headline        VARCHAR(200),           -- "Award-winning DP based in LA"
    resume_text     TEXT,
    credits         JSONB DEFAULT '[]',     -- [{title, role, year, production_company, imdb_url}]

    -- Rate card
    rate_min        DECIMAL(10,2),
    rate_max        DECIMAL(10,2),
    rate_unit       VARCHAR(10) CHECK (rate_unit IN ('hourly', 'daily', 'project')),
    rate_visibility VARCHAR(15) DEFAULT 'matches_only'
                    CHECK (rate_visibility IN ('public', 'matches_only', 'private')),
    rate_negotiable BOOLEAN DEFAULT TRUE,

    -- Preferences (what they're looking for when in client mode)
    seeking_roles   role_type[],            -- Roles they want to hire
    genre_tags      TEXT[],                 -- ['commercial', 'narrative', 'documentary', 'fashion']
    style_tags      TEXT[],                 -- ['cinematic', 'editorial', 'street', 'studio']

    completeness_score SMALLINT DEFAULT 0   -- 0-100, computed on update
                    CHECK (completeness_score >= 0 AND completeness_score <= 100),

    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### media_items

Photos and video reels uploaded by users.

```sql
CREATE TYPE media_type AS ENUM ('video_reel', 'photo', 'headshot');
CREATE TYPE processing_status AS ENUM ('pending', 'processing', 'ready', 'failed');
CREATE TYPE moderation_status AS ENUM ('pending', 'approved', 'rejected', 'flagged', 'appealed');

CREATE TABLE media_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID REFERENCES users(id) ON DELETE CASCADE,
    portfolio_id        UUID REFERENCES portfolios(id) ON DELETE CASCADE,

    media_type          media_type NOT NULL,
    original_url        TEXT NOT NULL,           -- S3 URL of original upload
    processed_url       TEXT,                    -- S3 URL of transcoded/optimized version
    thumbnail_url       TEXT,                    -- Auto-generated thumbnail

    -- Metadata
    title               VARCHAR(200),
    description         TEXT,
    duration_seconds    SMALLINT,                -- Video only
    width               SMALLINT,
    height              SMALLINT,
    file_size_bytes     BIGINT,
    mime_type           VARCHAR(50),

    -- Processing & moderation
    processing_status   processing_status DEFAULT 'pending',
    moderation_status   moderation_status DEFAULT 'pending',
    moderation_notes    TEXT,                    -- Reason for rejection/flag
    moderated_by        UUID,                   -- Admin who reviewed
    moderated_at        TIMESTAMPTZ,

    -- Ordering
    order_position      SMALLINT DEFAULT 0,
    is_primary          BOOLEAN DEFAULT FALSE,  -- Show first in portfolio

    created_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_media_user ON media_items (user_id, order_position);
CREATE INDEX idx_media_moderation ON media_items (moderation_status) WHERE moderation_status != 'approved';
CREATE INDEX idx_media_processing ON media_items (processing_status) WHERE processing_status != 'ready';
```

**Concern: Media Storage Costs**
- `file_size_bytes` tracked per item for billing/quota calculations
- `processing_status` allows lazy transcoding (only process when viewed)
- `moderation_status` gates public visibility — nothing shows until at least auto-moderated

### availability_slots

Calendar-based availability for talent.

```sql
CREATE TABLE availability_slots (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    date        DATE NOT NULL,
    status      VARCHAR(15) NOT NULL
                CHECK (status IN ('available', 'booked', 'unavailable', 'tentative')),
    visibility  VARCHAR(15) DEFAULT 'matches_only'
                CHECK (visibility IN ('public', 'matches_only', 'private')),
    notes       VARCHAR(200),
    UNIQUE (user_id, date)
);

CREATE INDEX idx_availability_user_date ON availability_slots (user_id, date);
CREATE INDEX idx_availability_date_status ON availability_slots (date, status);
```

### swipes

Records every swipe action. Core of the matching system.

```sql
CREATE TYPE swipe_direction AS ENUM ('like', 'pass', 'superlike');

CREATE TABLE swipes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    swiper_id       UUID REFERENCES users(id) ON DELETE CASCADE,
    swiped_id       UUID REFERENCES users(id) ON DELETE CASCADE,
    direction       swipe_direction NOT NULL,
    swiper_mode     VARCHAR(10) NOT NULL,       -- Mode user was in when they swiped
    created_at      TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE (swiper_id, swiped_id)               -- Can only swipe on someone once
);

-- Critical index for match detection: "has this person already liked me?"
CREATE INDEX idx_swipes_reverse_lookup
    ON swipes (swiped_id, swiper_id, direction)
    WHERE direction IN ('like', 'superlike');

-- Index for daily swipe count (rate limiting)
CREATE INDEX idx_swipes_daily_count
    ON swipes (swiper_id, created_at);
```

**Concern: Swipe Spam / Bot Prevention**
- `UNIQUE (swiper_id, swiped_id)` prevents double-swiping
- Daily count index enables efficient rate limit checks (100 swipes/day)
- `swiper_mode` recorded for audit trail if mode-switch abuse is suspected

### matches

Created when two users mutually like each other.

```sql
CREATE TABLE matches (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_a_id   UUID REFERENCES users(id) ON DELETE CASCADE,
    user_b_id   UUID REFERENCES users(id) ON DELETE CASCADE,
    status      VARCHAR(15) DEFAULT 'active'
                CHECK (status IN ('active', 'unmatched', 'blocked')),
    unmatched_by UUID,                          -- Who unmatched
    matched_at  TIMESTAMPTZ DEFAULT NOW(),

    -- Ensure user_a_id < user_b_id to prevent duplicate matches
    CHECK (user_a_id < user_b_id),
    UNIQUE (user_a_id, user_b_id)
);

CREATE INDEX idx_matches_user_a ON matches (user_a_id, status);
CREATE INDEX idx_matches_user_b ON matches (user_b_id, status);
```

### booking_requests

Project proposals from clients to talent (or between matched users).

```sql
CREATE TABLE booking_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requester_id        UUID REFERENCES users(id) ON DELETE CASCADE,
    target_id           UUID REFERENCES users(id) ON DELETE CASCADE,
    match_id            UUID REFERENCES matches(id),   -- NULL if sent without match

    -- Project details
    project_title       VARCHAR(200) NOT NULL,
    project_description TEXT NOT NULL,
    project_type        VARCHAR(50),                    -- 'commercial', 'film', 'photoshoot', etc.

    -- Logistics
    proposed_dates      JSONB,                          -- [{start, end}]
    location_description VARCHAR(200),
    proposed_rate       DECIMAL(10,2),
    rate_unit           VARCHAR(10),

    -- Status
    status              VARCHAR(15) DEFAULT 'pending'
                        CHECK (status IN ('pending', 'accepted', 'declined', 'withdrawn', 'expired')),
    response_message    TEXT,                            -- Target's reply
    responded_at        TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,                     -- Auto-expire after 7 days

    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_bookings_target ON booking_requests (target_id, status);
CREATE INDEX idx_bookings_requester ON booking_requests (requester_id, status);
CREATE INDEX idx_bookings_expiry ON booking_requests (expires_at) WHERE status = 'pending';
```

**Concern: Booking Request Spam (Without Match)**
- `match_id` being NULL means this is a "cold" request — these are rate-limited at the API layer
- `project_description TEXT NOT NULL` with a minimum length check (50 chars) at the API layer
- `expires_at` prevents stale requests from cluttering inboxes
- Requester's `trust_score` affects how many cold requests they can send per day

### follows

Social follow relationships for the public feed.

```sql
CREATE TABLE follows (
    follower_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    followed_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, followed_id)
);

CREATE INDEX idx_follows_followed ON follows (followed_id);  -- "who follows me"
```

### feed_posts

Content posted to the public scrollable feed.

```sql
CREATE TABLE feed_posts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID REFERENCES users(id) ON DELETE CASCADE,
    media_item_id       UUID REFERENCES media_items(id),    -- Optional linked media

    caption             TEXT,
    hashtags            TEXT[],

    -- Denormalized counters (updated async)
    like_count          INTEGER DEFAULT 0,
    comment_count       INTEGER DEFAULT 0,
    share_count         INTEGER DEFAULT 0,
    view_count          INTEGER DEFAULT 0,

    -- Moderation
    moderation_status   moderation_status DEFAULT 'pending',
    is_pinned           BOOLEAN DEFAULT FALSE,

    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_feed_posts_user ON feed_posts (user_id, created_at DESC);
CREATE INDEX idx_feed_posts_created ON feed_posts (created_at DESC)
    WHERE moderation_status = 'approved';
CREATE INDEX idx_feed_posts_hashtags ON feed_posts USING GIN (hashtags);
```

### feed_likes

```sql
CREATE TABLE feed_likes (
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    post_id     UUID REFERENCES feed_posts(id) ON DELETE CASCADE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);
```

### feed_comments

```sql
CREATE TABLE feed_comments (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    post_id     UUID REFERENCES feed_posts(id) ON DELETE CASCADE,
    parent_id   UUID REFERENCES feed_comments(id),  -- Threaded replies
    body        TEXT NOT NULL,

    moderation_status moderation_status DEFAULT 'approved',  -- Comments auto-approved, flagged retroactively

    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_comments_post ON feed_comments (post_id, created_at);
```

### messages

Real-time chat messages between matched users.

```sql
CREATE TABLE messages (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    match_id    UUID REFERENCES matches(id) ON DELETE CASCADE,
    sender_id   UUID REFERENCES users(id) ON DELETE CASCADE,

    content     TEXT NOT NULL,
    message_type VARCHAR(10) DEFAULT 'text'
                CHECK (message_type IN ('text', 'image', 'booking_link')),

    read_at     TIMESTAMPTZ,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_match ON messages (match_id, created_at);
CREATE INDEX idx_messages_unread ON messages (match_id, read_at) WHERE read_at IS NULL;
```

**Concern: Messaging Abuse**
- Messages only allowed between matched users (enforced by `match_id` FK)
- If either user unmatches, match status changes to `unmatched` and messaging is disabled
- Message content is not end-to-end encrypted (allows moderation of reported conversations)
- Image messages go through same moderation pipeline as media uploads

### notifications

```sql
CREATE TABLE notifications (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
    type        VARCHAR(30) NOT NULL,       -- 'new_match', 'new_message', 'booking_request', 'booking_response', 'new_follower'
    title       VARCHAR(200),
    body        TEXT,
    data        JSONB,                      -- Deep link data {screen, id}
    is_read     BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications (user_id, is_read, created_at DESC);
```

### reports

User-generated reports for content or behavior violations.

```sql
CREATE TYPE report_reason AS ENUM (
    'fake_profile', 'inappropriate_content', 'harassment',
    'spam', 'underage', 'scam', 'copyright', 'other'
);

CREATE TABLE reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reporter_id         UUID REFERENCES users(id) ON DELETE SET NULL,
    reported_user_id    UUID REFERENCES users(id),
    reported_post_id    UUID REFERENCES feed_posts(id),
    reported_message_id UUID REFERENCES messages(id),

    reason              report_reason NOT NULL,
    description         TEXT,
    evidence_urls       TEXT[],                 -- Screenshots uploaded by reporter

    -- Resolution
    status              VARCHAR(15) DEFAULT 'pending'
                        CHECK (status IN ('pending', 'reviewing', 'resolved', 'dismissed')),
    resolution          TEXT,
    resolved_by         UUID,
    resolved_at         TIMESTAMPTZ,

    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_reports_status ON reports (status, created_at);
CREATE INDEX idx_reports_target ON reports (reported_user_id, status);
```

**Concern: Report Abuse (Weaponized Reporting)**
- Multiple reports from the same reporter against the same user are deduplicated
- Reporter's own trust_score factors into report priority (low-trust reporters' reports are deprioritized)
- False reports (dismissed reports) reduce the reporter's trust_score
- Automated threshold: 3+ unique reporters on same target → auto-flag for priority review

### admin_actions (Audit Log)

```sql
CREATE TABLE admin_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id        UUID REFERENCES users(id),
    action_type     VARCHAR(30) NOT NULL,   -- 'suspend_user', 'approve_media', 'reject_media', 'resolve_report'
    target_user_id  UUID,
    target_item_id  UUID,
    reason          TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

## Migration Strategy

- Use **Alembic** for all schema migrations
- Each migration is versioned and reversible
- Seed data includes: role_type enum values, test users (dev only)
- PostGIS extension enabled via: `CREATE EXTENSION IF NOT EXISTS postgis;`
