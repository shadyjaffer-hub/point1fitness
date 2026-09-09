# Code Review Checklist

Run through this before merging any `feature/*` or `fix/*` branch into `main`. No merges without going through this — same weight as the RLS requirement.

## Before merging, confirm:

- [ ] **Matches the spec/scope** — does this actually do what `docs/scope.md` (or the feature spec) described? Nothing extra, nothing missing.
- [ ] **RLS in place** — if any table was added or changed, are Row Level Security policies defined and correct for it?
- [ ] **Migration included** — if the schema changed, is there a corresponding file in `supabase/migrations/`? No hand-edits via the Supabase dashboard.
- [ ] **No secrets committed** — no API keys, tokens, or credentials hardcoded or committed. `.env` respected.
- [ ] **No debug leftovers** — no stray `console.log`s, commented-out code blocks, or TODO markers left in without a reason.
- [ ] **Consistent with `CLAUDE.md`** — follows the established stack, folder structure, and conventions rather than introducing a new pattern ad hoc.
- [ ] **Builds and runs cleanly** — `npm run build` succeeds, no new errors/warnings introduced.
- [ ] **Tests present for logic changes** — if day-unlock, progress/streak, achievement triggers, or RLS behaviour changed, are there tests covering it? (UI-only changes don't need tests.)
- [ ] **Timestamps are `timestamptz` UTC** — no naive local timestamps introduced; timezone-sensitive logic uses the customer's profile timezone.
- [ ] **Server-side enforcement, not client-side** — any gating (day unlock, role access) is enforced in the database/RLS, not only hidden in the UI.
- [ ] **Validation is server-side too** — new inputs validated via the shared Zod schema, not client-only.
- [ ] **Accessibility basics** — touch targets ≥44px, form inputs labelled, keyboard navigable, sufficient contrast.
- [ ] **No eager YouTube iframes** — video embeds lazy-load on click.
- [ ] **Reviewed by a second pass** — either you personally reviewing the diff, or a fresh Claude Code session reviewing the branch against the spec (don't let the same session that wrote it be the only reviewer).

## How to use with Claude Code
Ask directly: *"Review the diff on this branch against `docs/scope.md` and this checklist before I merge. Flag anything missing."* Prefer a fresh session for this when possible — it won't share the same blind spots as the session that wrote the code.
