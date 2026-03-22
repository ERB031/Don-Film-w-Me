# UI/UX Design Options

Three design directions for Don Film w/ Me. Each option covers the same screens with different visual identities.

## Screen Inventory

| Screen | Purpose |
|--------|---------|
| **Swipe Cards** | Core matching — browse talent/clients |
| **Match Screen** | Celebration moment when mutual like happens |
| **Profile / Portfolio** | Talent showcase — reels, credits, rate card |
| **Feed (For You)** | Public content discovery |
| **Messaging** | Post-match chat |
| **Booking Request** | Send/receive project proposals |

---

## Option A: Dark & Cinematic

**Vibe**: Film set at golden hour. Dark backgrounds, amber/gold accents, cinematic typography.
**Inspiration**: IMDb Pro meets Bumble meets Netflix.

- Background: `#0D0D0D` (near-black)
- Primary accent: `#D4A843` (gold)
- Secondary: `#C75B39` (warm terracotta)
- Text: `#F5F0E8` (warm white)
- Card surface: `#1A1A1A`
- Font: Playfair Display (headings) + Inter (body)

**Strengths**: Premium feel, makes media pop against dark backgrounds, industry-appropriate.
**Weaknesses**: Can feel heavy, less approachable for casual users.

**Prototype**: `mockups/option-a-cinematic.html`

---

## Option B: Clean & Professional

**Vibe**: Modern SaaS meets talent marketplace. Light, airy, trustworthy.
**Inspiration**: LinkedIn meets Hinge meets Behance.

- Background: `#FAFAFA` (off-white)
- Primary accent: `#2563EB` (confident blue)
- Secondary: `#10B981` (success green)
- Text: `#1F2937` (dark gray)
- Card surface: `#FFFFFF` with subtle shadow
- Font: DM Sans (headings + body)

**Strengths**: Familiar, accessible, professional without being cold.
**Weaknesses**: Can feel generic, less "creative industry" energy.

**Prototype**: `mockups/option-b-professional.html`

---

## Option C: Bold & Creative

**Vibe**: Portfolio showcase meets festival poster. Vibrant, expressive, young.
**Inspiration**: Instagram meets Dribbble meets Letterboxd.

- Background: `#F8F5FF` (lavender tint)
- Primary accent: `#7C3AED` (electric violet)
- Secondary: `#F43F5E` (hot coral)
- Gradient: `#7C3AED` → `#F43F5E`
- Text: `#1E1B4B` (deep navy)
- Card surface: `#FFFFFF` with gradient border
- Font: Space Grotesk (headings) + Inter (body)

**Strengths**: Stands out, feels creative and energetic, appeals to younger talent.
**Weaknesses**: Can feel less "serious" for established industry professionals.

**Prototype**: `mockups/option-c-creative.html`

---

## ASCII Wireframes (Shared Layout)

All three options share the same information architecture and layout. The wireframes below show structure; the HTML prototypes show visual identity.

### Bottom Navigation

```
┌───────────────────────────────────────┐
│  🎬         🔍         📰         💬  │
│ Discover  Search     Feed       Chat  │
│                                  (2)  │
└───────────────────────────────────────┘
```

5 tabs: Discover (swipe), Search/Explore, Feed, Messages, Profile

---

### Screen 1: Swipe Cards

```
┌───────────────────────────────────────┐
│ [Mode: CLIENT ▼]          [Filters 🎛]│
├───────────────────────────────────────┤
│                                       │
│  ┌─────────────────────────────────┐  │
│  │                                 │  │
│  │                                 │  │
│  │        [ FULL-BLEED PHOTO       │  │
│  │          OR VIDEO REEL ]        │  │
│  │                                 │  │
│  │                                 │  │
│  │                                 │  │
│  │  ┌───────────────────────────┐  │  │
│  │  │ Sarah Chen, 28            │  │  │
│  │  │ DP / Cinematographer  ✓   │  │  │
│  │  │ 📍 Los Angeles • 3 mi    │  │  │
│  │  │ "Narrative & commercial   │  │  │
│  │  │  cinematography"          │  │  │
│  │  │                           │  │  │
│  │  │ [Commercial] [Narrative]  │  │  │
│  │  │ $800-1200/day             │  │  │
│  │  └───────────────────────────┘  │  │
│  │                                 │  │
│  │  ● ○ ○ ○  (media dots)         │  │
│  └─────────────────────────────────┘  │
│                                       │
│     ✕        ⭐        ♥             │
│    Pass    SuperLike    Like          │
│                                       │
│  Swipes: 87/100 remaining             │
├───────────────────────────────────────┤
│  🎬      🔍      📰      💬     👤   │
└───────────────────────────────────────┘
```

**Interaction**: Swipe right = like, left = pass. Tap card to expand full profile. Scroll within card to see more photos/reels. Buttons below for explicit tap.

---

### Screen 2: Expanded Profile Card (Tap on swipe card)

```
┌───────────────────────────────────────┐
│ [← Back to Discover]                  │
├───────────────────────────────────────┤
│                                       │
│  ┌─────────────────────────────────┐  │
│  │       [ HERO VIDEO REEL ]       │  │
│  │          ▶ 0:45 / 1:00          │  │
│  └─────────────────────────────────┘  │
│                                       │
│  Sarah Chen, 28               ✓ badge│
│  DP / Cinematographer                 │
│  📍 Los Angeles • Active today       │
│                                       │
│  ─── About ───                        │
│  Award-winning cinematographer with   │
│  5 years shooting narrative and       │
│  commercial projects across LA...     │
│                                       │
│  ─── Portfolio ───                    │
│  [img] [img] [img] [vid] [vid]       │
│  (horizontal scroll)                  │
│                                       │
│  ─── Credits ───                      │
│  • "Neon Dreams" (2025) — DP         │
│  • "East of Normal" (2024) — DP      │
│  • "Fresh Roast" (2024) — Camera Op  │
│  [See all 12 credits →]              │
│                                       │
│  ─── Rate Card ───                    │
│  $800 – $1,200 / day                 │
│  Negotiable ✓                         │
│                                       │
│  ─── Availability ───                 │
│  March: ██░░██████░░████░░████░░██   │
│  (green = available, gray = booked)   │
│                                       │
│  ─── Tags ───                         │
│  [Commercial] [Narrative] [Music Vid] │
│  [Cinematic] [Handheld] [Anamorphic] │
│                                       │
│     ✕        ⭐        ♥             │
│    Pass    SuperLike    Like          │
└───────────────────────────────────────┘
```

---

### Screen 3: Match Celebration

```
┌───────────────────────────────────────┐
│                                       │
│                                       │
│            ✨ IT'S A MATCH ✨         │
│                                       │
│         ┌──────┐    ┌──────┐         │
│         │ Your │    │Sarah │         │
│         │Avatar│ 💛 │Avatar│         │
│         └──────┘    └──────┘         │
│                                       │
│      You and Sarah Chen liked         │
│          each other!                  │
│                                       │
│   ┌─────────────────────────────┐    │
│   │      Send a Message         │    │
│   └─────────────────────────────┘    │
│                                       │
│   ┌─────────────────────────────┐    │
│   │     Send Booking Request    │    │
│   └─────────────────────────────┘    │
│                                       │
│         Keep Swiping →                │
│                                       │
└───────────────────────────────────────┘
```

---

### Screen 4: Feed (For You)

```
┌───────────────────────────────────────┐
│  [For You]  [Following]    [+ Post]   │
├───────────────────────────────────────┤
│                                       │
│  ┌─────────────────────────────────┐  │
│  │ (●) Marcus Rivera • Director ✓ │  │
│  │     Los Angeles • 2h ago       │  │
│  │                                 │  │
│  │  ┌───────────────────────────┐  │  │
│  │  │                           │  │  │
│  │  │    [ VIDEO REEL ]         │  │  │
│  │  │    ▶ 0:00 / 0:42         │  │  │
│  │  │                           │  │  │
│  │  └───────────────────────────┘  │  │
│  │                                 │  │
│  │  BTS from yesterday's music     │  │
│  │  video shoot 🎥                 │  │
│  │  #musicvideo #bts #director     │  │
│  │                                 │  │
│  │  ♥ 234    💬 18    ↗ 5         │  │
│  └─────────────────────────────────┘  │
│                                       │
│  ┌─────────────────────────────────┐  │
│  │ (●) Priya Patel • Model        │  │
│  │     New York • 5h ago          │  │
│  │                                 │  │
│  │  ┌───────────────────────────┐  │  │
│  │  │    [ PHOTO ]              │  │  │
│  │  └───────────────────────────┘  │  │
│  │                                 │  │
│  │  New editorial with @VogueMag   │  │
│  │  #editorial #fashion            │  │
│  │                                 │  │
│  │  ♥ 891    💬 42    ↗ 23        │  │
│  └─────────────────────────────────┘  │
│                                       │
├───────────────────────────────────────┤
│  🎬      🔍      📰      💬     👤   │
└───────────────────────────────────────┘
```

---

### Screen 5: Messaging

```
┌───────────────────────────────────────┐
│  Messages                      [Edit] │
├───────────────────────────────────────┤
│                                       │
│  ┌─── New Matches ───────────────┐   │
│  │ (●)(●)(●)(●)(●) →            │   │
│  │ Ava  Mo  Jay  Kim  Lee        │   │
│  └───────────────────────────────┘   │
│                                       │
│  ─── Conversations ───                │
│                                       │
│  ┌─────────────────────────────────┐  │
│  │ (●) Sarah Chen          2m ago │  │
│  │     Hey! Loved your reel...    │  │
│  │                         (unread)│  │
│  ├─────────────────────────────────┤  │
│  │ (●) Marcus Rivera       1h ago │  │
│  │     Sounds great, let me check │  │
│  │     my availability            │  │
│  ├─────────────────────────────────┤  │
│  │ (●) Priya Patel         3h ago │  │
│  │     📎 Booking Request         │  │
│  │     "Spring Campaign Shoot"    │  │
│  ├─────────────────────────────────┤  │
│  │ (●) Alex Kim            1d ago │  │
│  │     Thanks! Talk soon.         │  │
│  └─────────────────────────────────┘  │
│                                       │
├───────────────────────────────────────┤
│  🎬      🔍      📰      💬     👤   │
└───────────────────────────────────────┘
```

---

### Screen 6: Chat Thread

```
┌───────────────────────────────────────┐
│ [←]  Sarah Chen  ✓      [📋 Book]   │
├───────────────────────────────────────┤
│                                       │
│  ┌─ You matched on Mar 20 ─────────┐ │
│  └──────────────────────────────────┘ │
│                                       │
│                    ┌────────────────┐  │
│                    │ Hey Sarah! Love│  │
│                    │ your reel work │  │
│                    │ on Neon Dreams │  │
│                    └──── 10:30 AM ─┘  │
│                                       │
│  ┌────────────────┐                   │
│  │ Thanks! Are you│                   │
│  │ looking for DP │                   │
│  │ work? I saw    │                   │
│  │ your client    │                   │
│  │ profile        │                   │
│  └── 10:32 AM ────┘                   │
│                                       │
│                    ┌────────────────┐  │
│                    │ Yes! I have a  │  │
│                    │ music video    │  │
│                    │ shoot next     │  │
│                    │ month. Sending │  │
│                    │ you a booking  │  │
│                    │ request now    │  │
│                    └──── 10:35 AM ─┘  │
│                                       │
│  ┌──────────────────────────────────┐ │
│  │  📋 Booking Request Sent         │ │
│  │  "Spring Music Video — DP"       │ │
│  │  Apr 12-13 • $1,000/day          │ │
│  │  [View Details]                   │ │
│  └──────────────────────────────────┘ │
│                                       │
│  Sarah is typing...                   │
│                                       │
├───────────────────────────────────────┤
│  [   Type a message...    ] [Send]   │
└───────────────────────────────────────┘
```

---

### Screen 7: Booking Request (Send)

```
┌───────────────────────────────────────┐
│ [← Cancel]     New Booking    [Send] │
├───────────────────────────────────────┤
│                                       │
│  To: Sarah Chen (DP/Cinematographer) │
│  Via: Match 💛                        │
│                                       │
│  ─── Project Details ───              │
│                                       │
│  Project Title *                      │
│  ┌─────────────────────────────────┐  │
│  │ Spring Music Video              │  │
│  └─────────────────────────────────┘  │
│                                       │
│  Type                                 │
│  [Music Video ▼]                      │
│                                       │
│  Description *                        │
│  ┌─────────────────────────────────┐  │
│  │ Looking for a DP to shoot a     │  │
│  │ 3-min music video for indie     │  │
│  │ artist. Two-day shoot, mix of   │  │
│  │ narrative scenes and live       │  │
│  │ performance footage...          │  │
│  └─────────────────────────────────┘  │
│                                       │
│  ─── Dates ───                        │
│  ┌──────────────┐  ┌──────────────┐  │
│  │ Apr 12, 2026 │  │ Apr 13, 2026 │  │
│  └──────────────┘  └──────────────┘  │
│  Sarah's availability: ✅ Available   │
│                                       │
│  ─── Location ───                     │
│  ┌─────────────────────────────────┐  │
│  │ Downtown LA + Griffith Park     │  │
│  └─────────────────────────────────┘  │
│                                       │
│  ─── Rate ───                         │
│  Sarah's range: $800 – $1,200/day    │
│  ┌──────────┐  [Per Day ▼]           │
│  │ $1,000   │                         │
│  └──────────┘                         │
│  ✅ Within Sarah's listed range       │
│                                       │
│  ─── Expires in 7 days ───           │
│                                       │
└───────────────────────────────────────┘
```

---

### Screen 8: Profile / My Portfolio

```
┌───────────────────────────────────────┐
│  [⚙ Settings]              [Edit ✎] │
├───────────────────────────────────────┤
│                                       │
│         ┌──────────┐                  │
│         │  AVATAR  │                  │
│         └──────────┘                  │
│         Jordan Rivera                 │
│         Director / Producer  ✓        │
│         📍 Los Angeles                │
│         Active today                  │
│                                       │
│  ┌───────────┐ ┌───────────┐         │
│  │  TALENT   │ │  CLIENT   │         │
│  │  mode ●   │ │  mode ○   │         │
│  └───────────┘ └───────────┘         │
│                                       │
│  ─── Stats ───                        │
│  Matches: 24  │ Bookings: 8  │ ⭐ 4.8│
│                                       │
│  ─── Portfolio ───                    │
│  ┌──────┐ ┌──────┐ ┌──────┐         │
│  │ reel │ │ reel │ │ photo│  →      │
│  │  ▶   │ │  ▶   │ │      │         │
│  └──────┘ └──────┘ └──────┘         │
│  12 items • [Manage Portfolio →]     │
│                                       │
│  ─── Credits (8) ───                  │
│  • "Neon Dreams" (2025) — Director   │
│  • "Fresh Roast" (2024) — Producer   │
│  [See all →]                          │
│                                       │
│  ─── Rate Card ───                    │
│  $1,500 – $3,000 / day               │
│  Visibility: Matches only 🔒         │
│                                       │
│  ─── Availability ───                 │
│  March 2026                           │
│  ██░░██████░░████░░████░░████        │
│  [Manage Calendar →]                  │
│                                       │
│  ─── Trust Score ───                  │
│  ████████████████░░░░  82/100        │
│  Phone ✓  ID ✓  Portfolio ✓          │
│                                       │
├───────────────────────────────────────┤
│  🎬      🔍      📰      💬     👤   │
└───────────────────────────────────────┘
```
