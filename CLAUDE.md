# Point 1 Fitness — Claude Code Instructions

## What this is
A mobile-responsive gym web app for Point 1 Fitness, acting as a tool for coaches and customers to run and track fitness challenges (e.g. a 30-day challenge). Delivered as a **webview-wrapped web app via Capacitor** — NOT native. All core logic is web code (HTML/CSS/JS or a framework).

## Reference docs — read these for full detail
- Full scope: `/docs/scope.md`
- Full architecture: `/docs/architecture.md`
- Design direction & screen mockups: `/design/mockup.html`
- Code review checklist: `/docs/code-review-checklist.md`

Don't duplicate their content here — read them when you need the detail. This file only holds the facts that should apply to every session by default.

## Key facts to always keep in mind
- Not native — never suggest native iOS/Android code. Native tooling (Xcode/Android Studio) is only touched for the final build/sign/publish step via Capacitor.
- Three user flows: public marketing site (no login), customer flow (logged in), admin/coach flow (logged in, separate role)
- Daily challenge content unlocks **server-side**, gated by date/day-count — not just hidden in the UI
- Design tokens are already established (dark theme, orange brand accent `#f2711f`, Barlow Condensed/Barlow typefaces) — see `/design/mockup.html` and `/docs/architecture.md` for the full palette. Follow this direction rather than introducing a new visual style.

## Current build state
- Architecture is fully decided (see `/docs/architecture.md` decisions log) — stack: React + Vite + Capacitor + Supabase + YouTube video embeds
- No production app code written yet — this is the scaffolding starting point
- Design prototype exists: `/design/mockup.html` (static HTML/CSS design direction + screen gallery, not production code)

## Open questions — do not silently assume answers
- Can a customer join multiple challenges simultaneously, or one at a time?
- Does the loyalty/achievement mechanic persist across challenges, or reset each time?
- Is there a payment/subscription layer, or is access free once registered?

If a task depends on one of these, stop and ask rather than picking an answer.

## Branching & code review — mandatory, not optional
- Every feature/fix is done on its own branch (`feature/xxx`, `fix/xxx`, `chore/xxx`) — never commit directly to `main`
- Before any branch is merged to `main`, run through `/docs/code-review-checklist.md` in full. This includes confirming RLS policies are in place for any new/changed table, a migration file exists for any schema change, no secrets or debug leftovers are present, and the change matches the spec/scope it was built against.
- Prefer reviewing a branch from a **fresh session** rather than the same session that wrote it — a second pass catches what the original session's blind spots missed. If asked to review a branch, treat it as a real review: check it against the spec and the checklist, don't just confirm it runs.
- Database schema changes are always made via a tracked migration file in `supabase/migrations/`, never via hand-edits in the Supabase dashboard.

## Non-negotiable technical rules
These come from `/docs/architecture.md` and apply to every session. Do not deviate without raising it first.

- **Day-unlock is computed server-side**, anchored to the customer's local midnight of (enrollment date + N−1 days). The client may hide locked days for UX but must never be what enforces the gate.
- **All timestamps are `timestamptz` in UTC.** Customer timezone lives on their profile as an IANA string.
- **RLS on every table, no exceptions.** A new table without an RLS policy is an incomplete table.
- **Schema changes only via migration files** in `supabase/migrations/` — never hand-edits in the Supabase dashboard. Apply to `dev` first, then `prod`.
- **Soft deletes** (`deleted_at`) on any table customer history depends on. Never hard-delete a challenge or day.
- **Validation via shared Zod schemas** in `lib/`, enforced server-side. Client-side validation is UX, never a security control.
- **Server state goes through TanStack Query**, not hand-rolled `useEffect` fetching. Local UI state uses `useState`. No global state library without raising it first.
- **YouTube embeds are lazy-loaded** (thumbnail + play button until clicked), never rendered as full iframes on page load.
- **Tests are required** for day-unlock calculation, progress/streak computation, achievement triggers, and RLS behaviour. Not required for general UI rendering.
- **Never log secrets, tokens, or personal data** to console or Sentry.

## Working style
- Propose options with tradeoffs before implementing anything architecturally significant
- Keep `/docs/architecture.md`'s Key Architectural Decisions Log updated when a real decision gets made (date + rationale)
- Favor simplicity and clarity — this is a consumer-facing fitness product, not an internal tool
- Don't modify `/design/mockup.html` — treat it as a reference, not a file to build on top of directly
