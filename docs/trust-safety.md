# Trust & Safety

This document covers all identified concerns and their mitigations across the platform.

---

## 1. Fake Profiles & Identity Fraud

### The Concern
A matching platform is a magnet for fake profiles — catfishers, scammers, and bots pretending to be talent or clients to extract money, personal info, or simply waste people's time. In a professional context, fake casting directors or fake talent agencies pose additional risks.

### Mitigations

#### Tier 1: Baseline Verification (Required)
- **Phone number verification** via SMS OTP at signup (Twilio)
- One phone number per account (prevents mass account creation)
- VoIP/virtual numbers blocked (Twilio can detect these)
- Profile completeness gate: must have display name + at least 1 photo + bio (50+ chars) before entering the swipe queue

#### Tier 2: Enhanced Verification (Optional, Earns Badge)
- **Identity verification**: Upload government-issued photo ID → compared against selfie using facial recognition API (AWS Rekognition `CompareFaces`)
- Verified users get a badge on their profile card
- Verified users appear higher in swipe queues (10-point scoring bonus)
- Verification data is encrypted at rest, retained only until verification completes, then deleted (only the boolean `verified` flag persists)

#### Tier 3: Professional Verification (Future)
- **Portfolio verification**: Link IMDB, LinkedIn, or professional website → confirm account ownership
- **Industry vouching**: Verified users can vouch for others they've worked with
- **Agency verification**: Talent agencies can create verified agency accounts and link their talent

#### Automated Detection
- **Reverse image search**: On photo upload, run against known stock photo databases. Flag if match found.
- **Behavioral signals**: Flag accounts that:
  - Complete signup + swipe 100 profiles in < 10 minutes
  - Send identical booking request text to 5+ users
  - Switch modes > 5 times in 24 hours
  - Have 0 matches after 500+ swipes (likely bot or extremely spam-like behavior)

---

## 2. Content Moderation

### The Concern
User-uploaded photos and video reels need moderation for: NSFW content, violence, copyright infringement, and off-platform content (e.g., someone uploading random TikToks that aren't theirs).

### Moderation Pipeline

```
Upload → Virus Scan → Auto-Moderation → [Queue if flagged] → Human Review → Published
                           │
                           ├─ PASS → moderation_status = 'approved' → visible immediately
                           ├─ FLAG → moderation_status = 'flagged' → queued for human review
                           └─ REJECT → moderation_status = 'rejected' → user notified with reason
```

#### Tier 1: Automated Moderation (All Uploads)
- **NSFW detection**: AWS Rekognition `DetectModerationLabels` for images; frame sampling (1 frame/second) for video
- **Violence detection**: Same API, different label category
- **Face detection**: Ensure headshots actually contain a face
- **Text extraction (OCR)**: Detect phone numbers, emails, or URLs embedded in images (prevents contact info harvesting outside the platform)
- **Perceptual hashing**: Compare against known banned content hashes
- Processing time: < 5 seconds for photos, < 30 seconds for 60-second videos

#### Tier 2: Human Review Queue
- All auto-flagged content enters a review queue
- Target SLA: review within 24 hours (MVP), 4 hours (v1+)
- Reviewers can: approve, reject (with reason), or escalate
- All review actions logged in `admin_actions` table

#### Tier 3: Community Reporting
- Any user can report content via `POST /reports`
- Report reasons: inappropriate_content, harassment, spam, fake_profile, underage, scam, copyright, other
- Auto-escalation: 3+ unique reporters on same content → immediate temporary hide + priority review
- Reporter identity is never revealed to the reported user

#### Appeals Process
- Rejected content can be appealed once
- Appeal reviewed by a different moderator than the original reviewer
- Appeal decision is final
- Users with 3+ rejected uploads in 30 days get a warning; 5+ results in temporary upload suspension

### Content Rules (Published in App)
1. All media must be your own work or you must have rights to use it
2. Headshots/photos must show the actual person (no stock photos, no AI-generated faces)
3. Video reels must be relevant to your professional work
4. No nudity, violence, hate speech, or illegal content
5. No contact information (phone, email, social handles) embedded in media — use the platform's messaging
6. No watermarks from other platforms (prevents reuploading others' content)

---

## 3. Privacy & Data Protection

### The Concern
Users share sensitive professional information — rates, availability, location, portfolio — that needs granular privacy controls. Additionally, the platform must handle personal data (phone numbers, ID verification documents) responsibly.

### Privacy Controls (User-Facing)

| Data | Visibility Options | Default |
|------|--------------------|---------|
| Rate card (min/max) | Public / Matches only / Private | Matches only |
| Availability calendar | Public / Matches only / Private | Matches only |
| Exact location | Never shared | N/A |
| City/Region | Always visible | N/A |
| Portfolio | Always visible (public profile) | N/A |
| Phone number | Never shared publicly | N/A |
| Email | Never shared publicly | N/A |
| Last active time | Shown as "Active today/this week" (not exact) | N/A |

### Data Handling

- **Location**: Only city-level shown on profiles. Exact coordinates used only for distance calculations server-side, never sent to other clients.
- **EXIF stripping**: All uploaded photos have EXIF metadata removed (may contain GPS, device info).
- **ID verification documents**: Encrypted with AES-256 at rest. Deleted within 24 hours of verification completion. Only the verification result (pass/fail) is retained.
- **Password storage**: bcrypt with work factor 12. Never logged, never exposed in API responses.
- **API responses**: Never include `password_hash`, `phone`, `email`, or `location` coordinates when returning other users' data.

### Data Rights (GDPR/CCPA Compliant)

- **Data export**: Users can request a full export of their data (profile, media, swipe history, messages) via `DELETE /users/me` flow
- **Account deletion**: Soft delete (30-day recovery window), then hard delete of all personal data
- **Right to be forgotten**: On hard delete, user's data is anonymized in: swipes (swiper_id → null), messages (content → "[deleted]"), reports (reporter_id → null)
- **Consent**: Explicit opt-in for push notifications, location access, and marketing emails

### Blocking & Harassment Prevention

When User A blocks User B:
1. Match is dissolved (if any)
2. All messages between them are hidden from both
3. User B cannot see User A's profile anywhere (swipe queue, feed, search)
4. User B cannot send booking requests to User A
5. User B does not know they've been blocked (they simply stop seeing User A)
6. Block is bidirectional in effect but unilateral in action (A can unblock, B cannot)

---

## 4. Spam Prevention (Swipes, Messages, Bookings)

### The Concern
Without rate limits and quality signals, the platform could devolve into a spam-fest where low-effort interactions drown out genuine connections.

### Swipe Spam

| Control | Value | Purpose |
|---------|-------|---------|
| Daily swipe limit | 100 (normal), 30 (new accounts) | Prevents bot mass-swiping |
| SuperLike limit | 3/day | Preserves signal value |
| Spam-liker detection | >90% like ratio over 7 days → trust_score reduction | Discourages indiscriminate swiping |
| Minimum profile to swipe | completeness_score > 30 | Ensures swiper has a real profile |
| Swipe undo | 3/day, within 5 seconds | Allows correction, not gaming |

### Message Spam

| Control | Value | Purpose |
|---------|-------|---------|
| Messaging gate | Match required | No unsolicited DMs |
| Message rate limit | 60/minute per conversation | Prevents flooding |
| Link detection | URLs in messages trigger warning to recipient | Prevents phishing |
| Automated pattern detection | Flag accounts sending identical messages to 3+ matches | Catches copypasta spam |

### Booking Request Spam

| Control | Value | Purpose |
|---------|-------|---------|
| Cold request limit (no match) | 1-5/day based on trust_score | Trust-gated access |
| Minimum description length | 50 characters (cold requests) | Forces effort |
| Auto-expiry | 7 days | Prevents stale requests |
| Opt-out | Users can disable cold booking requests entirely | User control |
| Reputation tracking | Accepted requests improve trust_score; ignored/declined reduce it | Incentivizes quality requests |

---

## 5. Trust Score System

### The Concern
Need a way to differentiate trustworthy users from potentially problematic ones without relying solely on manual review.

### How Trust Score Works

Every user starts with a trust_score of 50 (range: 0-100).

**Score increases:**
| Action | Points | Cap |
|--------|--------|-----|
| Phone verified | +10 | Once |
| Identity verified | +20 | Once |
| Profile completeness > 80% | +5 | Once |
| First match | +5 | Once |
| Booking request accepted (by target) | +3 | Per event |
| 30 days without reports against you | +2 | Per period |
| Account age > 90 days | +5 | Once |

**Score decreases:**
| Action | Points | Cap |
|--------|--------|-----|
| Report filed against you (confirmed) | -10 | Per event |
| Content rejected by moderator | -5 | Per event |
| Spam-liker detection triggered | -5 | Per event |
| Booking request reported as spam | -10 | Per event |
| False report filed (you reported someone, dismissed) | -3 | Per event |
| Mode-switch abuse detected | -5 | Per event |

**Score thresholds:**
| Score Range | Effect |
|-------------|--------|
| 80-100 | Full access, higher visibility in queues, 5 cold requests/day |
| 60-79 | Normal access, 3 cold requests/day |
| 40-59 | Reduced visibility in queues, 1 cold request/day |
| 20-39 | Cannot send cold requests, must verify identity to restore access |
| 0-19 | Account suspended pending review |

---

## 6. Mode-Switching Abuse

### The Concern
The hybrid talent/client mode could be exploited — e.g., switching to client mode to see who's available, then switching back to talent mode to undercut their rates; or switching modes to circumvent swipe limits.

### Mitigations

1. **Cooldown**: Minimum 1 hour between mode switches
2. **Daily cap**: Maximum 5 mode switches per 24 hours
3. **Swipe history is mode-scoped**: Switching modes does NOT reset who you've already swiped on
4. **Rate limits are per-user, not per-mode**: Switching modes doesn't give you more daily swipes
5. **Audit trail**: All mode switches logged with timestamp, enabling pattern detection
6. **Anomaly detection**: Users who switch > 3x daily for 3+ consecutive days get flagged for review

---

## 7. Platform Safety (Beyond Content)

### Scam Prevention
- **Casting scam warning**: When a client's booking request mentions payment upfront, auditions at private locations, or requests personal photos outside the platform, auto-append a safety warning to the recipient
- **Rate card validation**: If a client proposes a rate that's < 20% of the talent's listed minimum, show a warning to both parties
- **Off-platform communication warning**: If messages contain phone numbers, email addresses, or social media handles within the first 5 messages of a conversation, show a safety tip encouraging users to keep communication on-platform until trust is established

### Physical Safety (Meeting in Person)
- **Share your plans**: Optional feature to share booking details (location, time, who you're meeting) with an emergency contact
- **Check-in system (v2)**: For in-person bookings, optional scheduled check-in. If user doesn't check in, emergency contact is alerted
- **Location sharing (v2)**: During an active booking, optional live location sharing with emergency contact

### Minor Safety
- **Age verification**: Users must confirm they are 18+ at signup
- **Report category**: Dedicated "underage" report reason with highest priority escalation (reviewed within 1 hour)
- **Zero tolerance**: Confirmed underage accounts are immediately suspended and reported to NCMEC if in the US

---

## 8. Moderation Team & Tooling

### MVP (Launch)
- 1-2 part-time moderators
- Admin dashboard with: report queue, flagged content queue, user trust score history
- Automated moderation handles 80%+ of content decisions
- Manual review for: flagged content, reported users, identity verification, appeals

### Scale (Post-Launch)
- Dedicated Trust & Safety team
- Tiered moderation (L1: content review, L2: account decisions, L3: legal/escalations)
- Moderator wellness program (exposure to harmful content)
- Regular audits of automated moderation accuracy (false positive/negative rates)
- Transparency reports (quarterly): number of reports, actions taken, appeal outcomes
