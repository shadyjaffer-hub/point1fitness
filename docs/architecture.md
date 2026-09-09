# Gym App — Architecture Reference

*This is a living document — update it as decisions are made during build.*

## Delivery Model
- Web app wrapped in **Capacitor** for mobile app store distribution (iOS + Android)
- Not native — all core logic is web code (HTML/CSS/JS or a framework)
- Public marketing pages should also work as a standard responsive website (not just inside the wrapped app)

## Tech Stack
- **Frontend framework:** **React** (confirmed) — best fit for the app's complexity (daily unlock logic, progress tracking, role-based views) and strongest Capacitor pairing + Claude Code proficiency
- **Build tool:** **Vite** (confirmed) — standard modern pairing with React, fast dev server, clean handoff to `npx cap sync`
- **Styling:** Design direction established (see `/design/mockup.html`):
  - Dark theme — bg `#141312`, raised surfaces `#1c1a18`/`#211f1c`, text `#f6f3ee`
  - Brand accent: orange `#f2711f` (with darker variant `#c65102`), matching the Point 1 logo
  - Status colors: green `#4ac47a` (complete/success), amber `#e0a736` (in-progress)
  - Typography: **Barlow Condensed** (display/headings) + **Barlow** (body), via Google Fonts
- **Backend:** **Supabase** (confirmed) — Postgres-based, fits the app's relational data (Challenge → Days → Workouts/Nutrition, Customer → DailyLogs) far better than a NoSQL alternative; built-in auth with role support covers the customer/coach split; auto-generated APIs reduce custom backend work
- **Database:** **Postgres, via Supabase**
- **Push notifications (future, if needed):** Supabase has no native push service — would require a Postgres trigger → Edge Function → Firebase Cloud Messaging/APNs pipeline. Not needed for v1; revisit if daily reminders become a priority feature.
- **Video hosting:** **YouTube (unlisted embeds)** (confirmed for now) — free, no real downside at launch scale. Build as a single reusable `VideoPlayer` component that takes a video reference (ID/URL) rather than hardcoding embeds per-screen, so switching to Vimeo or another provider later is a low-effort swap, not a rebuild.
- **Auth:** **Supabase Auth**, method: **email + password** for v1 (both customer and coach/admin roles). Social login (Google/Apple) planned as a fast-follow for the customer signup flow to reduce friction — not v1, not architecturally blocking. Magic link not used as primary method (poor fit for a daily-check-in app), may be offered as a secondary option later.

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

## Project Structure
Folder structure mirrors the three user flows directly:
```
src/
  routes/
    public/     — marketing site, no auth (Home, ChallengeList, CaseStudies, Team, Login)
    customer/   — logged-in customer flow (Today, DayDetail, Workout, Nutrition, Progress, Achievements, Profile)
    admin/      — coach/admin panel (Customers, Challenges, Workouts, Nutrition, Leads)
  components/   — shared, reusable UI
  layouts/      — per-flow page shells (PublicLayout, CustomerLayout, AdminLayout)
  theme/        — design tokens as real CSS/JS, extracted from /design/mockup.html
  lib/          — API clients, auth helpers, utilities
  hooks/        — custom React hooks
```
`ios/` and `android/` are Capacitor-generated — not hand-edited directly.

## Local Dev Workflow
- **Day-to-day:** `npm run dev` (Vite dev server, hot reload, browser-based) — use for the large majority of UI work
- **Native device checks:** `npm run build` → `npx cap sync` → `npx cap run ios` / `npx cap run android` — reserve for milestones (end of a feature, pre-release), not every edit, since it's slower than the browser loop

## Environment & Config Management
- `.env` (git-ignored) for local secrets/config; `.env.example` committed with placeholder keys to document required variables
- Vite requires frontend-exposed vars to be prefixed `VITE_` (e.g. `VITE_API_URL`)
- Concrete variables TBD until backend/auth/video hosting are decided

## CI
- Minimal for now: GitHub Actions workflow running `npm install && npm run build` on every push — just catches broken builds
- Revisit (tests, deployment) once real code and a test suite exist — not urgent pre-build

## Key Architectural Decisions Log
*(Add entries here as they're made, with date and rationale)*

| Date | Decision | Rationale |
|---|---|---|
| 2026-09-09 | Frontend framework: React | Best fit for app complexity (unlock logic, progress state, role-based views); strongest Capacitor pairing; strongest Claude Code proficiency |
| 2026-09-09 | Build tool: Vite | Standard modern React pairing, fast dev server, clean Capacitor integration |
| 2026-09-09 | Folder structure: routes split by public/customer/admin | Mirrors the three user flows from scope.md directly, keeps flow-specific code isolated |
| 2026-09-09 | Local dev via `npm run dev`; native checks only at milestones | Faster iteration loop; native build/sync is too slow for every edit |
| 2026-09-09 | Env config via `.env` + `.env.example`, `VITE_`-prefixed vars | Standard Vite pattern; keeps secrets out of git while documenting required vars |
| 2026-09-09 | CI: build-check only on push (GitHub Actions) | Minimal safety net appropriate for pre-build stage; expand once code/tests exist |
| 2026-09-09 | Backend/database: Supabase (Postgres) | Relational data model fits challenge/workout/progress structure; built-in auth with roles; better admin/reporting query support than a NoSQL alternative; push notifications (if needed later) will require bridging to FCM/APNs |
| 2026-09-09 | Video hosting: YouTube (unlisted embeds) | Free, no real downside at launch scale; low switching cost later if built as one reusable player component |
| 2026-09-09 | Auth: Supabase Auth, email+password for v1 | Simple to build first, works for both roles; magic link poor fit for daily-check-in app; social login planned as fast-follow to reduce customer signup friction |

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
