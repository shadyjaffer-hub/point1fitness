# Gym App — Architecture Reference

*This is a living document — update it as decisions are made during build.*

## Delivery Model
- Web app wrapped in **Capacitor** for mobile app store distribution (iOS + Android)
- Not native — all core logic is web code (HTML/CSS/JS or a framework)
- Public marketing pages should also work as a standard responsive website (not just inside the wrapped app)

## Tech Stack
- **Frontend framework:** TBD (React / Vue / other) — design prototype currently built in static HTML/CSS
- **Styling:** Design direction established (see `/design/mockup.html`):
  - Dark theme — bg `#141312`, raised surfaces `#1c1a18`/`#211f1c`, text `#f6f3ee`
  - Brand accent: orange `#f2711f` (with darker variant `#c65102`), matching the Point 1 logo
  - Status colors: green `#4ac47a` (complete/success), amber `#e0a736` (in-progress)
  - Typography: **Barlow Condensed** (display/headings) + **Barlow** (body), via Google Fonts
- **Backend:** TBD (needed for: auth, challenge/progress data, admin content management, leads storage)
- **Database:** TBD
- **Video hosting:** TBD (self-hosted vs. YouTube/Vimeo embeds for movement demo videos)
- **Auth:** TBD (needed for customer login/register + separate admin/coach access level)

## High-Level System Structure

### Public Site
- Static/marketing pages (landing, case studies, team, leads form)
- No auth required
- Leads form submits to backend for admin visibility

### Customer App (post-login)
- Challenge dashboard (current challenge progress)
- Daily unlock logic — server-side gated by date/day-count, not just UI-hidden
- Workout viewer (per day, per movement, with video links)
- Nutrition plan viewer (per day)
- Daily achievement logging
- Loyalty/achievement display

### Admin/Coach Panel
- Challenge management (create/edit challenge structure, days, content)
- Workout + nutrition content management per challenge day
- Customer progress visibility
- Leads management
- Team/case study content management

## Native Shell (Capacitor)
- Build pipeline: `npm run build` → `npx cap sync` → `npx cap open ios` / `npx cap open android`
- Native tooling needed only for build/test/publish steps (Xcode for iOS, Android Studio for Android)
- No native plugins anticipated yet — evaluate if push notifications (e.g. daily reminders) are needed later

## Key Architectural Decisions Log
*(Add entries here as they're made, with date and rationale)*

| Date | Decision | Rationale |
|---|---|---|
| — | — | — |

## Design Reference
Full design direction and screen mockups: `/design/mockup.html`. Covers foundations (color/type tokens), public homepage, and member + coach/admin screen flows:
- **Public:** Hero & CTA, Challenge list (card gallery), Log in
- **Member:** Today, Day detail, Workout overview, Video & movement, Workout complete, Today's meals, Meal breakdown, Daily check-in, Progress, Achievements, Profile
- **Coach/admin:** Customers, Challenges, Workouts, Nutrition, Leads

**Note:** the public Challenge list is designed as a browsable card gallery for visitors to pick a challenge before logging in — suggests multiple challenges are meant to be offered/selectable, though this doesn't by itself confirm whether one customer can be enrolled in more than one simultaneously (still open, see below).

## Open Technical Questions
- Multi-challenge support per customer (impacts data model) — design leans toward challenges being browsable/selectable, but simultaneous enrollment is still unconfirmed
- Loyalty mechanic persistence across challenges (impacts data model)
- Payment/subscription layer (impacts auth + backend scope significantly if added)
