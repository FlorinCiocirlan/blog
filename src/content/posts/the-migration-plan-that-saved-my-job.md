---
title: "The migration plan that saved my job"
description: "Two AngularJS-to-Angular migrations had failed before I got there. The third worked. Here is the phase plan, per-screen checklist, and feature-freeze mechanics — exactly as we ran it for 5,000+ users."
pubDate: 2026-05-27
tags: ["angular", "angularjs", "migration", "tech-lead", "engineering-leadership"]
readingTime: "7 min read"
draft: false
---

Two years ago I was six weeks from quitting this job.

The frontend was AngularJS. The team had tried to migrate twice. Both attempts died in the same place: a "big bang" rewrite that estimated 18 months and ran out of patience by month 4.

I came in as the new senior. I read both post-mortems. The pattern was the same in both:

1. A small group built a "v2" in parallel.
2. v2 caught up to about 30% feature parity.
3. The business asked when it would be done.
4. The engineering team couldn't answer.
5. The work stopped.

Same mistake, twice: nobody had a phase plan. Nobody had defined what "done" looked like for a single screen, let alone the whole product.

I almost wrote the resignation email.

Instead, I asked for one week to redesign the plan from scratch.

## What I changed

### 1. Reframed "migration" as "feature freeze + parallel build"

The previous attempts thought of the work as a *replacement project*. That framing is poisonous. It implies: do nothing useful for X months, then flip a switch.

Reframe:

- **AngularJS app:** maintenance-only. No new features. Bug fixes on a strict allow-list (security, payment, blocker-only).
- **Angular 16 app:** every new feature ships here. The product roadmap continues, just in the new codebase.

The legacy app stops accumulating debt. The new app accumulates value. Migration becomes the *natural consequence* of feature work, not a project that competes with it.

### 2. Defined "migrated" per screen, not per app

This was the single biggest change.

Every screen got a one-page migration spec:

```
SCREEN: contract-discounts-list
LEGACY ROUTE: /angular-js/#/contracts/:id/discounts
NEW ROUTE: /contracts/:id/discounts (Angular 16, behind feature flag)

DONE WHEN:
  [ ] Route mounted in new app
  [ ] All API calls migrated to typed services (HttpClient + Observable)
  [ ] State managed via NgRx feature module (where shared) OR local component signals
  [ ] AG Grid configured with shared column types
  [ ] Reactive forms with typed FormGroup
  [ ] All user actions covered by e2e tests
  [ ] Feature flag toggled ON for internal users for 1 sprint
  [ ] Feature flag toggled ON for 10% of customers for 1 sprint
  [ ] Flag removed, legacy route returns 301 to new route
  [ ] Legacy controller/template DELETED in same PR
```

The crucial line is the last one. **Done means the old code is gone.** No "we'll clean it up later." Later never comes.

### 3. Cut the surface area

This was the unpopular call.

The legacy app had ~140 distinct screens. About 60% of them carried 95% of the revenue. The other 40% were:

- Admin screens used by 2-3 internal users
- Reports that had been replaced by Tableau dashboards but never deleted
- A "v2" attempt by a previous team that was half-finished and half-broken
- Settings pages nobody opened

We didn't migrate them. We listed them, got product sign-off, and let them rot inside the AngularJS shell. When the last revenue-generating screen migrated, we killed the AngularJS app and everything in it died with it.

This freed something like 6 months of engineering time. Migrating to migrate is a tax on the future.

### 4. Made engineering standards part of the migration deliverable

The other migrations had failed partly because the new code looked exactly like the old code, just in a different framework. Same anti-patterns, same God services, same untestable forms.

We made the standards the gate:

- **Standalone components by default** once we were on Angular 14+; **SCAM modules** (Single Component Angular Modules — one component per module, lazy-loadable) as the bridge pattern while the new app was still on the older baseline. Same isolation guarantees either way.
- **State: pick the smallest tool that fits.** A service exposing `signals()` (or a `BehaviorSubject<State>` on the older versions) carries most route-local and feature-local state. **NgRx only when state is genuinely shared across feature boundaries** — usually 20-30% of features at most. Don't NgRx everything; the store becomes the new God service the moment you do.
- **Typed reactive forms** with `FormGroup<{}>` interfaces — no `any` in form value
- **Shared column types** for AG Grid (price, quantity, date, percent) — defined once, used everywhere
- **One service per resource**, returning Observables, no Promises in the new app
- **No PR merge** without a passing e2e test for the migrated user action

If a PR didn't meet the bar, it didn't ship. Even if it meant the migrated screen was slower than the legacy one. The point of the new app wasn't to be a clone — it was to be a foundation.

## What happened

- First migrated screen shipped in 3 weeks. (The contract-discounts list, ~4,000 line items in AG Grid, under 1s render.)
- The whole product followed in waves over ~18 months.
- 5,000+ users never saw a flag day. Each screen flipped behind a flag, validated for a sprint at 10%, then 100%.
- The team stopped having to defend a quarter-long rewrite to leadership.
- The AngularJS bundle shrank to zero. The Angular bundle shrank ~50% from the optimizations we did *after* the migration was structurally sound.

I never sent the resignation email.

## The thing I learned

When a project has failed twice, the third plan can't be a better version of the same plan. It has to redefine what "shipping" means.

The previous attempts defined shipping as *the new app reaching parity with the old app*. That's a goal you can't see from month 1, and that's why teams burn out before they reach it.

The plan that worked defined shipping as *one screen, with all its dependencies, replaced and deleted*. That's a goal you can hit in 3 weeks. Hit it 20 times in a row and you've done the migration.

The framing is the work.

## What I'd do differently

A few things, in retrospect:

- **Spec the feature flag infrastructure on day one.** We retrofitted ours in week 4 and it cost us a sprint.
- **Pick the loudest screen first**, not the easiest one. The team needs an early win that customers can see, not a backend rewrite nobody notices.
- **Pair the engineer who built the legacy version with the one rewriting it.** Tribal knowledge lives in the legacy author's head; the rewrite is 2x faster when they pair for the first day.
- **Document the deletion ritual.** When a screen is fully migrated, the team gets to delete the legacy file together in a PR. It sounds silly. It's the single most motivating ceremony I've ever installed.

## If you're in this position

Read both of your previous post-mortems. Find the line where the engineering team couldn't answer "when will it be done." That's the line your new plan has to fix.

A migration plan is not a Gantt chart. It's a definition of *done* small enough to ship in 3 weeks, repeated 20-50 times.

That's the third plan.

---

*If you're looking at an AngularJS-to-Angular migration (or any "we tried this twice and it died") and want a sanity check on your plan, my inbox is open: ciocirlanflorinld@gmail.com.*
