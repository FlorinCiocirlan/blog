---
title: "The LEGACY-7: a 7-step audit before you say yes to an AngularJS migration"
description: "The exact checklist I run before agreeing to lead a legacy-Angular migration. If your team can't answer all 7, the migration will die at month 4 the same way the last two did."
pubDate: 2026-05-28
updatedDate: 2026-05-28
tags: ["angular", "angularjs", "migration", "tech-lead", "architecture", "legacy-7"]
draft: false
---


I charge engineering teams to audit their AngularJS-to-Angular migration plan. Today I'm publishing the checklist for free.

It's called the **LEGACY-7**. Seven questions, in order. If you can't answer all seven honestly with evidence in your repo, your migration will die at month 4 the same way the last two did. I know because I've now sat next to three teams that died at exactly that month, and they failed on the same questions.

Block 45 minutes. Open your project in one tab, this article in the other. Grade yourself ruthlessly. If you want a second opinion, the offer at the bottom of the page is real.

---

## Why most Angular migrations fail

I've been on three migrations (one I led, two I joined as the senior who came in after the last attempt died). The death cycle looks the same:

1. A small group builds "v2" in parallel.
2. v2 catches up to ~30% feature parity.
3. The business asks when it will be done.
4. The engineering team can't answer.
5. The work stops.

The pattern in every post-mortem: nobody defined what "done" looked like for a single screen, let alone the whole product. The plan was "rewrite the app." Plans like that can't survive contact with a roadmap.

The LEGACY-7 is what I changed.

---

## 1. Define "DONE" per screen, not per app

"The new app reaches feature parity" is a 12-month goal you can't see from month 1. That's why your team burned out before reaching it.

A screen is **done** when:

- [ ] Route mounted in the new app
- [ ] All API calls migrated to typed services (HttpClient + Observable, no Promise)
- [ ] State managed via NgRx feature module (where shared) OR local component signals
- [ ] Forms migrated to typed reactive forms (`FormGroup<{}>`)
- [ ] AG Grid / table component configured with shared column types
- [ ] All user actions covered by an e2e test that passes against the new app
- [ ] Feature flag flipped ON for internal users for 1 sprint
- [ ] Feature flag flipped ON for 10% of customers for 1 sprint
- [ ] Feature flag removed, legacy route returns 301 to new route
- [ ] **Legacy controller/template/service DELETED in the same PR**

The last line is non-negotiable. *Done means the old code is gone.* "We'll clean it up later" — later never comes. You will be migrating dead code for the rest of your career.

A "done" you can hit in 3 weeks, repeated 30 times, is a migration. A "done" you can only hit at the end is a death march.

---

## 2. Cut the surface area

Audit your routes. Sort them by revenue contribution.

On my last migration, ~60% of routes carried ~95% of revenue. The remaining 40% was:

- Admin screens used by 2-3 internal users
- Reports that had been replaced by Tableau dashboards but never deleted
- A "v2" attempt by a previous team, half-finished and half-broken
- Settings pages nobody opened
- A migration tool nobody had needed in 18 months

**We didn't migrate them.** We listed them, got product sign-off, and let them die with the legacy shell.

This freed roughly 6 months of engineering time. Migrating to migrate is a tax on the future.

The unpopular part: get product to put their name on the kill list. "Engineering decided to drop these" is a fight. "Product approved the deprecation list" is a decision.

---

## 3. Freeze features on the legacy app. Ship new features only in the new app.

The previous migrations failed because they thought of the work as a *replacement project*. That framing is poisonous. It implies: do nothing useful for X months, then flip a switch.

Reframe:

- **Legacy app:** maintenance-only. Bug fixes on a strict allow-list — security, payment, data-loss only. Everything else: closed as wontfix or moved to the new app's backlog.
- **New app:** every new feature ships here. The product roadmap continues; it just continues in the new codebase.

The legacy stops accumulating debt. The new app accumulates value. Migration becomes the *natural consequence* of feature work, not a project that competes with it for headcount.

Selling this is easier than it sounds. Product cares about velocity, not framework choice. "Every new feature ships faster in the new app" is a roadmap argument, not a technical one.

---

## 4. Engineering standards are part of the deliverable

The previous migration failed partly because the new code looked exactly like the old code, just in a different framework. Same God services. Same untestable forms. Same `any` everywhere.

We made the standards the gate. No PR merges without:

- **Standalone components by default** if you're on Angular 14+ — they remove the module boilerplate that was the biggest accidental coupling source in the legacy app. **SCAM modules** (Single Component Angular Modules — one component per module, lazy-loadable) are the right bridge pattern if you're stuck on an older version: same isolation guarantees as standalone, expressed in the syntax the version supports.
- **Pick the smallest state tool that fits.** A service with `signals()` carries most route-local and feature-local state in a modern Angular app. A service exposing a `BehaviorSubject<State>` does the same on older versions. **NgRx (or NGXS, Akita, Elf) only when state is genuinely shared across feature boundaries** and the trace/replay devtools earn their weight — usually no more than 20-30% of features. Don't NgRx everything; the store becomes the new God service the moment you do.
- **Typed reactive forms** — `FormGroup<{ field: FormControl<string> }>` — no `any` in form value
- **One service per resource**, returning Observables, no Promises in the new app
- **Shared column types for AG Grid** — price, quantity, date, percent — defined once, used everywhere
- **Cypress (or Playwright) e2e tests** for every user action on the migrated screen
- **No PR merge** without a passing e2e and a deleted legacy file

If a PR doesn't meet the bar, it doesn't ship. Even if it means the migrated screen ships a week later. The point of the new app isn't to be a faster clone of the legacy — it's to be the foundation for the next decade.

---

## 5. Feature flags on day 1, not week 4

Build the feature flag infrastructure before you migrate a single screen.

I retrofitted ours in week 4 and we lost a full sprint to it — backfilling flag wrappers around already-migrated routes, rewriting the rollout tests, rebuilding the targeting config. Every screen migrated before the flags existed had to be re-validated against the flag system.

Day 1 setup, in order:

1. A flag service that reads from a config source (LaunchDarkly, Unleash, or a JSON file if you're cheap)
2. A targeting layer — by user ID, by company ID, by percentage rollout
3. A route guard that resolves which app (legacy or new) renders a given route based on flag state
4. A monitoring dashboard that shows flag exposure and error rates per flag
5. A rollback runbook — one button, one config change, no deploy

This is 1-2 weeks of work before any migration starts. It feels like overhead. It is. It will save you 4-8 weeks across the migration.

---

## 6. Pick the loudest screen first, not the easiest

The instinct is to migrate the simplest screen first — a settings page, a static list, something with no state.

Don't. The team needs an early win **customers see**, not a backend rewrite nobody notices.

The dashboard. The pricing page. The thing the CEO opens on Monday morning. The screen sales demos to prospects. Migrate that first. The momentum from a visible win pays for the next 3 quiet ones.

The technical argument for this: the loud screens are usually the *hardest* ones, so you find the architectural problems early — when you still have 18 months to fix them. Migrating settings pages first means you discover that AG Grid doesn't fit your new state model in month 11. That's not a discovery you want at month 11.

---

## 7. Pair the legacy author with the rewriter on day 1

Tribal knowledge lives in the original author's head. The rewrite is 2x faster when they pair on day 1.

What pairing accomplishes that documentation can't:

- "Why does this component subscribe to four observables?" — the comment will not tell you. The author will.
- "What's that 200-line if-block actually doing?" — usually one production incident from 2021 that everyone forgot.
- "Is this field nullable?" — the schema says no. Production data says yes. Author confirms in 10 seconds.
- "What happens if you click Save twice fast?" — there's a debounce somewhere. Author knows where.

After day 1, the rewriter owns it. Day 1 is just to download the head-state.

If the original author already left: read the `git blame` on the screen. Find whoever code-reviewed those PRs. They know more than the comments do. Buy them coffee. Ask the same questions.

---

## What you do with this

Run it on your project. Score yourself 1-7. Two failure modes are the most common:

- **Step 1 (no per-screen done):** the team is working hard but can't tell you what shipping looks like
- **Step 5 (no flags from day 1):** the team is one bad deploy away from a rollback that takes hours

Fix those two and your migration's probability of shipping roughly doubles in my experience.

The other five matter, but they matter less. You can recover a migration that fails 6 and 7. You can't recover one that fails 1 and 5.

---

## Want a second opinion on your plan?

If you're sitting on an AngularJS-to-Angular plan and want a sanity check, send it over. Architecture doc, wiki page, even a Loom walkthrough — no code needed. I read every one and reply with the 2-3 LEGACY-7 steps I'd worry about first.

Reach me on LinkedIn: [Florin Ciocirlan](https://www.linkedin.com/in/florinciocirlan-angular/)
Or email: ciocirlanflorinld@gmail.com

---

*Companion piece: [The migration plan that saved my job](/posts/the-migration-plan-that-saved-my-job/) — the full story of the migration where the LEGACY-7 was built.*
