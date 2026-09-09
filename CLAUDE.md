# Point 1 Fitness — Claude Code Instructions

## What this is
A mobile-responsive gym web app for Point 1 Fitness, acting as a tool for coaches and customers to run and track fitness challenges (e.g. a 30-day challenge). Delivered as a **webview-wrapped web app via Capacitor** — NOT native. All core logic is web code (HTML/CSS/JS or a framework).

## Reference docs — read these for full detail
- Full scope: `/docs/scope.md`
- Full architecture: `/docs/architecture.md`
- Design direction & screen mockups: `/design/mockup.html`

Don't duplicate their content here — read them when you need the detail. This file only holds the facts that should apply to every session by default.

## Key facts to always keep in mind
- Not native — never suggest native iOS/Android code. Native tooling (Xcode/Android Studio) is only touched for the final build/sign/publish step via Capacitor.
- Three user flows: public marketing site (no login), customer flow (logged in), admin/coach flow (logged in, separate role)
- Daily challenge content unlocks **server-side**, gated by date/day-count — not just hidden in the UI
- Design tokens are already established (dark theme, orange brand accent `#f2711f`, Barlow Condensed/Barlow typefaces) — see `/design/mockup.html` and `/docs/architecture.md` for the full palette. Follow this direction rather than introducing a new visual style.

## Current build state
- Planning / early scoping phase — no production app code written yet
- Design prototype exists: `/design/mockup.html` (static HTML/CSS design direction + screen gallery, not production code)
- Frontend framework, backend, database, video hosting, and auth are all **still TBD** — don't assume a stack; ask or flag it if a decision is needed to proceed

## Open questions — do not silently assume answers
- Can a customer join multiple challenges simultaneously, or one at a time?
- Does the loyalty/achievement mechanic persist across challenges, or reset each time?
- Is there a payment/subscription layer, or is access free once registered?

If a task depends on one of these, stop and ask rather than picking an answer.

## Working style
- Propose options with tradeoffs before implementing anything architecturally significant
- Keep `/docs/architecture.md`'s Key Architectural Decisions Log updated when a real decision gets made (date + rationale)
- Favor simplicity and clarity — this is a consumer-facing fitness product, not an internal tool
- Don't modify `/design/mockup.html` — treat it as a reference, not a file to build on top of directly
