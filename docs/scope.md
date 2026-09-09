# Gym App — Scope Reference

## Overview
A mobile-responsive gym web app, built as a webview-wrapped web app (Capacitor, not native), acting as a tool for coaches and customers to run and track fitness challenges (e.g. a 30-day challenge).

## User Flows

### 1. Public / Marketing Site (no login required)
- Landing page with case studies and top transformations
- Team section
- Basic leads capture form
- Entry point into the challenge section, with login/register flow

### 2. Customer Flow (logged in)
- Daily challenge tracking (e.g. 30-day format) — customer logs daily achievement
- Workout schedule view, each movement linked to a demonstration video
- Daily nutrition plan
- Days unlock progressively, one per day (not all visible upfront)
- Basic loyalty/achievement mechanic tied to progress through the challenge

### 3. Admin/Coach Flow
- Manage challenges, workout schedules, nutrition plans, and video links
- Manage/view customer progress
- Manage leads captured from the public form
- Manage team/case study content shown on the public site

## Design Priority
Must be genuinely user-friendly — this is a consumer-facing fitness product, not an internal tool. Favor simplicity and clarity over feature density.

## Non-Functional Requirements
"User-friendly" as a stated priority needs measurable targets to be actionable:

- **Performance:** interactive within ~3s on a mid-range Android device over 4G. Workout pages must not load a video player per movement (thumbnails until clicked).
- **Accessibility:** WCAG 2.1 AA. Minimum 44×44px touch targets — this is used mid-workout, with sweaty hands and imprecise taps.
- **Device/browser support:** iOS 15+, Android 10+, current Chrome/Safari/Firefox/Edge for the public site.
- **Reliability:** a check-in must never appear to succeed when it hasn't saved. Failure states are explicit and recoverable.
- **Data safety:** a customer's logged progress must survive coach-side content edits and deletions.

## Open Items — To Be Defined During Scoping

### Previously identified
- Whether customers can join multiple challenges simultaneously
- Whether the loyalty mechanic persists across challenges or resets each time
- Whether there's a payment/subscription layer, or access is free once registered

### Surfaced during architecture review (2026-09-09)
- **Missed-day policy.** If a customer misses day 5, does day 6 unlock anyway (calendar-driven) or do they stay on day 5 until it's completed (progress-driven)? This directly shapes the unlock logic and the meaning of "streak" — it cannot be deferred past the first build of the challenge flow.
- **Content authoring burden.** A 30-day challenge is 30 days × workouts × movements × nutrition plans. Entering each day manually is punishing for a coach. Needs a duplication/templating mechanism (copy a previous challenge, copy a day, reusable movement library) — otherwise the admin panel is technically complete but practically unusable.
- **Enrollment path.** Does a customer self-enroll from the public challenge list, or does a coach assign/approve them? Affects both the public flow and the enrollment data model.
- **Coach vs admin.** This doc treats them as one flow. Are they genuinely one role, or two with different permissions (e.g. a coach sees only their own customers; an admin manages team/case-study content and all customers)? Affects the role enum and every RLS policy.

## Assets
- Gym logo — [to be added to project knowledge base separately]

## Status
Planning / early scoping phase.
