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

### Core frontend libraries (confirmed)
Decided upfront so they aren't picked ad hoc mid-feature, which is how codebases end up with three different patterns for the same job.

- **React Router** — routing across the three flows (public / customer / admin), with route guards enforcing role access
- **TanStack Query** — all server state (fetching, caching, refetching, optimistic updates) against Supabase. Prevents hand-rolled `useEffect` + loading/error state scattered across every screen.
- **React Hook Form + Zod** — forms and validation. Zod schemas live in `lib/` and are shared between client-side validation and server-side checks, so a rule is defined once rather than duplicated and drifting.

Rule of thumb: local UI state → React `useState`. Anything that comes from or goes to the database → TanStack Query. No global state library (Redux/Zustand) unless a concrete need appears — the two categories above cover this app.

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

## Data Model

Covers everything structurally certain. The two relationships that genuinely depend on the open scope questions are flagged inline.

### Conventions (apply to every table)
- **UUID primary keys** (`id`), not sequential integers — avoids leaking record counts and makes future data merging safe
- **`snake_case`** table and column names (Postgres convention, keeps Supabase's generated APIs predictable)
- **`created_at` / `updated_at`** as `timestamptz`, always UTC, on every table
- **`created_by`** referencing `profiles.id` on any coach-authored content, for traceability
- **Soft deletes** (`deleted_at timestamptz NULL`) rather than hard deletes on any table a customer's history depends on — a coach deleting a challenge must never destroy customers' logged progress
- **RLS enabled on every table** with no exceptions (see Database Change Management)

### Entities

**`profiles`** — extends Supabase `auth.users` (Supabase owns credentials; this holds app data)
- `id` (FK → `auth.users.id`), `full_name`, `avatar_url`
- `role` — enum: `customer` | `coach` | `admin` (see open question on coach vs admin)
- `timezone` — IANA string (e.g. `Africa/Cairo`), captured at signup. Drives day-unlock (see Timezone rules).

**`challenges`** — a challenge program (e.g. "30-Day Shred")
- `title`, `description`, `duration_days`, `difficulty`, `cover_image_url`
- `status` — `draft` | `published` | `archived`. Drafts let a coach build a 30-day challenge before exposing it publicly.

**`challenge_days`** — one row per day of a challenge
- `challenge_id` (FK), `day_number`, `title`, `notes`
- Unique constraint on (`challenge_id`, `day_number`)

**`workouts`** — the workout attached to a challenge day
- `challenge_day_id` (FK), `title`, `description`

**`movements`** — individual exercises within a workout
- `workout_id` (FK), `name`, `sets`, `reps`, `rest_seconds`, `order_index`
- `video_provider` (`youtube` for now) + `video_ref` (the ID/URL). Storing provider separately is what makes the later switch to Vimeo a data migration rather than a rewrite.

**`nutrition_plans`** — the nutrition plan for a challenge day
- `challenge_day_id` (FK), `notes`, `total_calories` (nullable)

**`meals`** — meals within a nutrition plan
- `nutrition_plan_id` (FK), `name` (e.g. Breakfast), `description`, `order_index`

**`enrollments`** — a customer taking a challenge. **This is the table the open questions hit.**
- `profile_id` (FK), `challenge_id` (FK), `started_at` (`timestamptz`), `status` (`active` | `completed` | `abandoned`)
- **Open:** if only one active challenge is allowed per customer, add a partial unique index on (`profile_id`) where `status = 'active'`. If simultaneous enrollment is allowed, omit it. Deliberately left unconstrained until decided — adding the constraint later is a trivial migration; removing a wrong assumption baked into application logic is not.

**`daily_logs`** — a customer's check-in for a given day
- `enrollment_id` (FK), `challenge_day_id` (FK), `completed_at`, `notes`
- **Unique constraint on (`enrollment_id`, `challenge_day_id`)** — makes check-ins idempotent, so a double-tap or flaky connection retry can't create duplicate records

**`achievements`** — earned loyalty/achievement records
- `profile_id` (FK), `type`, `earned_at`, `enrollment_id` (**nullable**)
- **Open:** nullable `enrollment_id` deliberately supports both answers to the loyalty question — populated if achievements are per-challenge, left null if they're lifetime/cross-challenge. Costs nothing to keep both doors open.

**`leads`** — public form submissions
- `name`, `email`, `phone`, `message`, `source`, `status` (`new` | `contacted` | `converted`)
- Insert-only for anonymous users via RLS; readable only by coach/admin

**`team_members`** and **`case_studies`** — public marketing content managed from the admin panel
- Both: `title`/`name`, `body`/`bio`, `image_url`, `order_index`, `published` (boolean)

### Relationship summary
```
profiles ──< enrollments >── challenges
                 │                │
                 │                └──< challenge_days ──< workouts ──< movements
                 │                          │
                 │                          └──< nutrition_plans ──< meals
                 └──< daily_logs >── challenge_days
profiles ──< achievements
```

## Timezone & Day-Unlock Rules

The core mechanic of the app depends on "what day is it for this customer" — a classic source of subtle, user-visible bugs. Rules, decided upfront:

- **Store everything in UTC** using `timestamptz`. Never store naive local timestamps.
- **Capture each customer's IANA timezone** on their profile at signup (e.g. `Africa/Cairo`), and let them change it — people travel and relocate.
- **Day N unlocks at the customer's local midnight of (`enrollment.started_at` date + N−1 days).** Anchored to their enrollment, not the shared calendar, so a customer joining at 11pm still gets a full first day rather than four minutes of it.
- **Unlock is computed server-side** — a Postgres function plus RLS policy. The client may hide locked days for UX, but must never be the thing enforcing it. A UI-only gate is trivially bypassed.
- **Check-ins are idempotent** — enforced by the unique constraint on `daily_logs`. Retries and double-taps are safe by construction rather than by careful client code.
- **Timezone changes must not retroactively invalidate past check-ins.** Completed days stay completed; only future unlock boundaries shift.

## Environments

Two environments, both on free tiers — chosen deliberately to stay at $0 given resource constraints:

- **`dev`** — a separate Supabase project. All development and testing runs here. Vercel preview deployments (per branch/PR) point at this database.
- **`prod`** — a separate Supabase project holding real customer data. Only the `main` branch deploys here.

Rules:
- **Migrations are applied to `dev` first, verified, then to `prod`.** Never the other way round.
- **Never hand-edit `prod`** via the Supabase dashboard — same rule as schema changes generally.
- Each environment has its own Supabase URL/keys, supplied via environment variables (`.env` locally, Vercel environment settings for deployments).

## Testing Strategy

Deliberately minimal and targeted — resources are limited, so testing effort goes where bugs are both most likely and most costly, not toward a coverage percentage.

- **Tooling:** **Vitest** (native Vite pairing, no extra config burden) + **React Testing Library** for component-level tests
- **What gets tested (non-negotiable):**
  - **Day-unlock calculation** — timezone edge cases, enrollment-anchored day boundaries, DST transitions
  - **Progress and streak computation** — the logic behind what a customer sees as their achievement
  - **Achievement trigger conditions** — when a badge is/isn't earned
  - **RLS policies** — that a customer genuinely cannot read another customer's logs
- **What is not tested:** general UI rendering, styling, layout. Eyeballing catches these faster than tests do, and chasing UI coverage is where limited resources get burned for little return.
- **E2E testing (Playwright or similar): deferred.** Revisit only when a specific recurring integration bug justifies the setup cost.

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
- **Seed data:** a committed `supabase/seed.sql` populating a realistic sample challenge (30 days with workouts, movements, nutrition), plus test customers at varying progress points — including one mid-challenge (e.g. day 12) and one who has missed days. Without this, a fresh environment starts empty and screens like "Day 12 detail" or streak displays can't meaningfully be tested at all.

## Environment & Config Management
- `.env` (git-ignored) for local secrets/config; `.env.example` committed with placeholder keys to document required variables
- Vite requires frontend-exposed vars to be prefixed `VITE_` (e.g. `VITE_API_URL`)
- Concrete variables TBD until backend/auth/video hosting are decided

## CI
- Minimal for now: GitHub Actions workflow running `npm install && npm run build` on every push — just catches broken builds
- Revisit (tests, deployment) once real code and a test suite exist — not urgent pre-build

## Branching & Version Control Strategy
- **Trunk-based + feature branches:** `main` stays always-deployable; every feature or fix gets its own branch, merged via PR
- **Naming convention:** `feature/xxx`, `fix/xxx`, `chore/xxx`
- **Goal:** each branch/PR maps to one discrete change — enables clean single-revert rollback and clear traceability in git history, supporting long-term maintainability (fix/enhance without destabilizing the whole app)

## Database Change Management
- **Supabase migrations, not dashboard hand-edits.** Every schema change (tables, columns, RLS policies) is written as a migration file in `supabase/migrations/`, committed to the repo alongside the feature branch that needs it
- This keeps the database's state traceable and revertable through git history, the same way code changes are — critical given the maintainability goal (easy fixes/enhancements on a solid base)
- RLS (Row Level Security) policies are mandatory on every table from the start — without them, the Supabase auto-generated API allows any authenticated user to read/write other users' data

## Hosting (frontend/public deployment)
- **Proposed: Vercel** — zero-config deploy from GitHub, free tier sufficient at this stage, automatic SSL, preview deployments per branch/PR (useful given the branching strategy above), pairs commonly with Supabase
- **Not yet confirmed** — revisit alongside CI decision, since Vercel's per-push build may make the separate GitHub Actions build-check redundant

## Error Handling, Logging & Monitoring

Without this, the first you hear about a production bug is a customer emailing you — if they bother.

- **React Error Boundaries per route section** (public / customer / admin) so one broken screen degrades gracefully instead of white-screening the whole app
- **Sentry** (free tier) for error tracking in production — captures the stack trace and context you'd otherwise be guessing at
- **One shared error-handling helper in `lib/`** rather than ad hoc `try/catch` blocks with inconsistent messaging scattered through components
- **User-facing errors are actionable** — say what happened and what to do, never a raw error string or a silent failure. Especially critical for check-in failures: a customer must never believe they logged a day that didn't save.
- **Never log secrets, tokens, or personal data** to console or to Sentry

## Database Backups

Supabase's free tier backup coverage is thin, and customer progress data is emotionally valuable — losing someone's 30-day streak is a trust-killer that no apology fixes.

- **Stopgap while on free tier:** scheduled `pg_dump` via a GitHub Action, output stored off-platform (not in the same account as the database)
- **Explicit upgrade trigger:** move to a paid Supabase tier with proper point-in-time recovery **as soon as real customer progress data exists** — this is not a "when we can afford it" item, it's a launch prerequisite
- Test a restore at least once before launch. An untested backup is a hope, not a backup.

## Analytics

Needed because the whole product premise is engagement, and without measurement there's no way to know if it's working.

- **PostHog** (free tier) — chosen over page-view tools for funnel and retention analysis, which is what actually matters here
- **Key events to instrument:** signup, challenge enrollment, daily check-in, **drop-off day** (the day a customer stops returning — the single most valuable number in this product), challenge completion, video play
- **Privacy:** analytics must respect the privacy policy; no personal data in event properties; disclose in the privacy policy

## Performance

Video- and image-heavy on mobile, often on poor connections.

- **Route-based code splitting** — the public marketing site shouldn't ship the admin panel's JavaScript to a first-time visitor
- **Lazy-load YouTube iframes until clicked** — render a lightweight thumbnail + play button instead of a full embed. A workout page with eight movements would otherwise load eight YouTube players at once, which is a serious mobile performance cost. This is the highest-impact single optimization in the app.
- **Responsive, optimized images** for transformation photos and case studies — these are the heaviest assets on the marketing site
- Keep an eye on Capacitor bundle size; audit before each release

## Accessibility

Target: **WCAG 2.1 AA**. Cheaper to build in than to retrofit.

- **Minimum 44×44px touch targets** — this is a mobile app used mid-workout, with sweaty hands and imprecise taps
- Semantic HTML, proper heading hierarchy, labelled form inputs
- Full keyboard navigability and visible focus states
- Meaningful `alt` text on images; captions/transcripts considered for movement videos
- **Action item:** verify the brand orange `#f2711f` against the dark background `#141312` for contrast. It likely passes for large display text but **may fail AA for body text** — if so, define a lighter accent variant for small text rather than compromising the brand colour on headings.
- Respect `prefers-reduced-motion`

## Abuse Protection

The leads form is a public, unauthenticated endpoint — these get spammed.

- **Honeypot field** (invisible input that only bots fill) — free, no friction for real users
- **Cloudflare Turnstile** (free) if the honeypot proves insufficient — less hostile to users than traditional CAPTCHAs
- **Rate limiting** on public endpoints
- **Server-side validation via the shared Zod schemas** — client-side validation is a UX affordance, never a security control
- RLS on `leads`: anonymous users may insert only; read access restricted to coach/admin

## Signing & Publishing (future, pre-launch)
- **iOS:** Apple Developer account, signing certificate, provisioning profile — set up in Xcode at publish time
- **Android:** signing keystore — generated once, must be kept safe permanently (losing it blocks future updates to the same store listing)
- Not urgent now; flagged so it's not a surprise at first submission

## Code Review
- No branch merges to `main` without going through `docs/code-review-checklist.md` — same non-negotiable weight as the RLS requirement
- Review combines: self-review of the diff, checking against the spec/scope, and a fresh Claude Code session reviewing the branch (avoids the same blind spots as the session that wrote the code)
- Full checklist: `docs/code-review-checklist.md`

## Privacy & Legal (future, pre-launch)
- Privacy policy and Terms of Service required for both App Store and Play Store submission
- Data being collected is health/fitness-adjacent (workouts, nutrition, progress logs) — worth extra care in what's collected/retained
- Not an architecture task — legal drafting, but flagged here so it's not forgotten

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
| 2026-09-09 | Branching: trunk-based + feature branches | One branch/PR per discrete change enables clean rollback and traceability; lighter than Git Flow for this team size |
| 2026-09-09 | DB changes via tracked Supabase migrations, not dashboard edits | Keeps schema history traceable/revertable in git, matching the code branching discipline; supports long-term maintainability goal |
| 2026-09-09 | RLS mandatory on all Supabase tables | Prevents cross-user data access via the auto-generated API; non-negotiable baseline, not optional hardening |
| 2026-09-09 | Mandatory code review checklist before merge, incl. fresh-session Claude Code review | No large team available for review; a repo-stored checklist + a second Claude Code session catches issues the original session/self-review alone would miss |
| 2026-09-09 | Data model defined upfront, with open questions isolated to two nullable/unconstrained fields | Prevents Claude Code inventing a schema mid-feature; schema mistakes are the most expensive kind to unwind. Both open questions deliberately kept answerable later without a rewrite. |
| 2026-09-09 | Day-unlock anchored to customer's local midnight of (enrollment date + N−1), computed server-side | Enrollment-anchored (not calendar-anchored) gives every customer a full first day; server-side computation prevents trivial client-side bypass; UTC storage + IANA profile timezone is the standard correct pattern |
| 2026-09-09 | Idempotent check-ins via unique constraint on (enrollment_id, challenge_day_id) | Makes double-taps and retries safe by construction rather than by careful client code |
| 2026-09-09 | Two environments (dev + prod), separate Supabase projects | Prevents testing against real customer data and bad migrations hitting production; both fit free tier, so $0 cost |
| 2026-09-09 | Testing: Vitest + RTL, scoped to unlock/progress/achievement logic and RLS only | Limited resources — testing effort targeted at logic that is costly to get wrong and hard to eyeball, rather than a coverage percentage. E2E deferred until justified. |
| 2026-09-09 | Frontend libraries: React Router, TanStack Query, React Hook Form + Zod | Decided upfront so they aren't picked ad hoc mid-feature; Zod schemas shared client/server so validation rules are defined once. No global state library until a concrete need appears. |
| 2026-09-09 | Soft deletes on history-bearing tables | A coach deleting a challenge must never destroy customers' logged progress |
| 2026-09-09 | Sentry (free) + per-route Error Boundaries | Production errors otherwise only surface via customer complaints; boundaries prevent one broken screen white-screening the app |
| 2026-09-09 | Scheduled pg_dump backups now; paid Supabase tier as a launch prerequisite | Free-tier backup coverage is insufficient for data as trust-critical as a customer's 30-day streak |
| 2026-09-09 | PostHog (free) for analytics, drop-off day as key metric | Engagement is the product premise; funnel/retention analysis matters more than page views |
| 2026-09-09 | Lazy-load YouTube iframes until clicked | Highest-impact single performance win — a workout page would otherwise load one full player per movement on mobile |
| 2026-09-09 | Accessibility target WCAG 2.1 AA, 44px touch targets | Mobile app used mid-workout with imprecise taps; far cheaper to build in than retrofit. Brand orange needs a contrast check for body text. |
| 2026-09-09 | Leads form: honeypot + server-side Zod validation, Turnstile if needed | Public unauthenticated endpoints get spammed; client-side validation is UX, never a security control |
| 2026-09-09 | Committed seed data (`supabase/seed.sql`) | Fresh environments otherwise start empty, making mid-challenge screens (day 12, streaks) untestable |

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
