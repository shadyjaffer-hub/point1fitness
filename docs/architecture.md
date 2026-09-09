# Gym App — Architecture Reference

*This is a living document — update it as decisions are made during build.*

## Delivery Model
- Web app wrapped in **Capacitor** for mobile app store distribution (iOS + Android)
- Not native — all core logic is web code (HTML/CSS/JS or a framework)
- Public marketing pages should also work as a standard responsive website (not just inside the wrapped app)

## Tech Stack
- **Frontend framework:** TBD (React / Vue / other)
- **Styling:** TBD
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

## Open Technical Questions
- Multi-challenge support per customer (impacts data model)
- Loyalty mechanic persistence across challenges (impacts data model)
- Payment/subscription layer (impacts auth + backend scope significantly if added)
