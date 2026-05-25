---
title: "How I Saved the Company Thousands Every Day With These Changes"
description: "Four concrete Angular optimizations that slashed build size by 43%, cut CI/CD costs, and improved time to first byte for 5,000+ users."
pubDate: 2026-05-25
tags: ["angular", "performance", "scam", "bundle-optimization", "frontend"]
draft: false
---

Every millisecond of load time costs money.

Our Angular app was shipping 5.3MB of JavaScript on every cold start. CI/CD builds were eating minutes and cloud compute. Production deployments were slow. Dev environment deploys were slow. 5,000+ users were waiting on every page load.

Nobody flagged it as a crisis. It was just... the way it was.

I decided to change that.

## The Starting Point

Before touching anything, I opened [source-map-explorer](https://github.com/danvk/source-map-explorer) and looked at what was actually inside the bundle.

```bash
ng build --source-map
npx source-map-explorer dist/your-app/**/*.js
```

What I found was predictable once you see it, but easy to miss when you're shipping features every sprint:

- A few massive feature modules with nothing lazy loaded
- Moment.js taking 72KB for date formatting we could do with 6KB
- AG Grid's entire module set imported, despite using roughly a third of it
- Angular Material's `MatDrawer` eagerly loaded on every route

Four problems. Four fixes.

## Fix 1: Split Modules and Enforce SCAM

Our app had a handful of large feature modules — the kind that grow over time because dropping a new component into an existing module is always easier than creating a new one.

The problem: Angular can only lazy load at the module level. If your module is large, users download all of it on first visit regardless of which route they land on.

We were on **Angular 13** at the time — before standalone components existed. Standalone arrived in Angular 14 and became the recommended default in Angular 17. In v13, NgModules were the only option, which made module discipline critical.

I started splitting modules by responsibility. One feature, one module. Then introduced **SCAM** (Single Component Angular Modules) as our default pattern going forward.

SCAM means every component gets its own dedicated module:

```typescript
// Before: everything piled into one feature module
@NgModule({
  declarations: [InvoiceListComponent, InvoiceDetailComponent, InvoiceFormComponent],
  imports: [CommonModule, FormsModule, MaterialModule],
})
export class InvoiceModule {}

// After: each component owns its module
@NgModule({
  declarations: [InvoiceListComponent],
  imports: [CommonModule, MatTableModule, MatSortModule],
  exports: [InvoiceListComponent],
})
export class InvoiceListModule {}
```

This makes lazy loading surgical. A route only loads the exact components it needs, nothing more.

The part that makes this stick long-term: I made SCAM a PR review requirement. Every new component, every PR — if it's not following SCAM, it doesn't merge. Without that enforcement layer, the modules just grow back.

If you're on Angular 14+ today, standalone components give you the same granularity natively. SCAM was the pre-standalone equivalent — same discipline, different API.

## Fix 2: Moment.js → Day.js (via a DateService)

Source map explorer flagged Moment.js immediately. 72KB minified, just for date formatting.

Day.js is API-compatible with Moment.js and weighs 6KB.

```bash
npm uninstall moment
npm install dayjs
```

Rather than doing a raw find-and-replace across the entire codebase, I created a `DateService` utility that wraps Day.js. Every date operation in the app goes through it:

```typescript
@Injectable({ providedIn: 'root' })
export class DateService {
  format(date: Date | string, format = 'DD/MM/YYYY'): string {
    return dayjs(date).format(format);
  }

  fromNow(date: Date | string): string {
    return dayjs(date).fromNow();
  }

  isBefore(date: Date | string, reference: Date | string): boolean {
    return dayjs(date).isBefore(reference);
  }

  isAfter(date: Date | string, reference: Date | string): boolean {
    return dayjs(date).isAfter(reference);
  }

  diff(dateA: Date | string, dateB: Date | string, unit: dayjs.OpUnitType = 'day'): number {
    return dayjs(dateA).diff(dateB, unit);
  }
}
```

This approach has a few advantages over find-and-replace:

- **Single import point** — Day.js is imported once. If you switch libraries again in the future, you change one file.
- **Controlled plugin surface** — only the Day.js plugins you actually need get registered, in one place.
- **Testable** — mocking `DateService` in tests is trivial, no global date mocking needed.

Done in one afternoon. 66KB saved, and the codebase is better structured for it.

## Fix 3: AG Grid Per-Module Imports

AG Grid is a powerful library. It's also a large one.

We were importing the entire thing:

```typescript
import { AgGridModule } from 'ag-grid-angular';

@NgModule({
  imports: [AgGridModule],
})
export class MyModule {}
```

That pulls in the full AG Grid package. We were using sorting, filtering, and basic cell rendering. That's it.

The fix is per-module imports — you only pay for what you actually use:

```typescript
// In your component
import { ClientSideRowModelModule, SortModule, FilterModule } from 'ag-grid-community';

@Component({
  template: `<ag-grid-angular [modules]="modules" ...></ag-grid-angular>`
})
export class MyGridComponent {
  modules = [ClientSideRowModelModule, SortModule, FilterModule];
}
```

Check AG Grid's documentation for the exact module names matching your version — the API changed significantly between v27, v28, and v30+.

## Fix 4: Lazy Load the Angular Material Drawer

This one is subtle and I haven't seen it documented much anywhere.

Angular Material's `MatDrawer` and `MatDrawerContainer` are eagerly loaded by default. The moment `MatSidenavModule` appears anywhere in your app, that code is in the initial bundle — whether or not the user ever opens a drawer.

We use drawers heavily: detail panels, filter panels, action panels across multiple features. The cost was real.

I built a `DrawerService` that lazy loads the drawer component only when it's first opened:

```typescript
@Injectable({ providedIn: 'root' })
export class DrawerService {
  private drawerRef: ComponentRef<DrawerComponent> | null = null;

  async open(content: Type<unknown>, data?: unknown): Promise<void> {
    if (!this.drawerRef) {
      // Drawer module is NOT in the initial bundle
      const { DrawerComponent } = await import('./drawer/drawer.component');
      // dynamically create and attach to the DOM
    }
    this.drawerRef.instance.open(content, data);
  }

  close(): void {
    this.drawerRef?.instance.close();
  }
}
```

The dynamic `import()` is the key. The drawer chunk is fetched the first time `DrawerService.open()` is called — not before. A user who lands on a read-only view and never opens a drawer never downloads that code.

## The Results

| Metric | Before | After |
|---|---|---|
| Bundle size | 5.3 MB | 3.0 MB |
| Build time | slow | ~2x faster |
| Time to first byte | baseline | noticeably improved |
| CI/CD compute per build | baseline | significantly reduced |

43% smaller bundle.

CI/CD resource costs dropped across every PR, every deployment, every engineer pushing code throughout the day. That compounds fast in an active team.

For a 5,000+ user platform, the time to first byte improvement is real money. Page speed affects retention and infrastructure cost simultaneously.

And crucially: these wins hold. Every new component added under SCAM stays lean. Every lazy route stays clean. The PR review process prevents the bundle from drifting back to where it was.

## Where to Start

If you haven't run source-map-explorer on your Angular build yet, do that first. You'll know in 10 minutes where your biggest wins are hiding.

```bash
ng build --source-map
npx source-map-explorer dist/your-app/**/*.js
```

Then work through this checklist:

1. Find your Moment.js equivalent — almost every app has one
2. Audit your large library imports (AG Grid, Chart.js, etc.) for tree-shakeable alternatives
3. Introduce SCAM as a PR standard, not just a one-time refactor
4. Identify any Angular Material modules loaded eagerly that could be deferred
5. Use `--source-map` on every build until the bundle feels right

The build is a balance sheet. Every kilobyte has a cost. Start reading it.
