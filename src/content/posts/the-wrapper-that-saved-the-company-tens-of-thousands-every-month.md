---
title: "The wrapper that saved the company tens of thousands every month"
description: "How I turned repeated AG Grid implementations into one typed Angular wrapper that cut delivery time, reduced QA churn, and made a complex dashboard platform easier to scale."
pubDate: 2026-05-21
updatedDate: 2026-05-21
tags: ["angular", "typescript", "architecture", "ag-grid", "frontend", "developer-experience"]
readingTime: "9 min read"
draft: false
---

> The highest-leverage frontend work I have done was not a shiny new screen.
>
> It was a wrapper.

Not a wrapper in the superficial sense.

Not a thin component that hides a library because someone does not like its API.

I mean a real product-level abstraction: one that removed repeated work, reduced defects, made QA faster, standardised the user experience, and saved the company tens of thousands every month.

The funny part is that users never saw the wrapper directly.

They saw dashboards that behaved consistently.

Developers saw a clean interface.

QA saw fewer duplicated behaviours to validate.

The business saw delivery become faster and less expensive.

That is the kind of frontend architecture I care about.

## The situation I walked into

When I joined the company, one of the core products had a data-grid-heavy dashboard experience.

These were not simple tables.

They were enterprise dashboards with serious behaviour:

- AG Grid
- grouping
- filtering
- searching
- editing
- column formatters
- charts
- persisted layouts
- state restoration
- custom interactions
- domain-specific business rules

The dashboard was important for the business because users relied on it to analyse and operate on data every day.

So the grid experience had to be powerful.

But the implementation model had a problem.

Every developer was copy-pasting grid setup from another screen.

A new feature needed a grid, so someone would find the closest existing dashboard, copy the configuration, remove some columns, add new ones, tweak callbacks, wire state persistence, add formatting, and hope nothing important was missed.

At first glance, that does not sound catastrophic.

The screens worked.

The product moved forward.

But underneath, the company was paying a quiet tax every sprint.

## The hidden cost of copy-paste architecture

The business problem was not AG Grid.

AG Grid is powerful. It gives teams many of the primitives needed to build sophisticated data experiences: column definitions, events, formatting, state APIs, charts, grouping, filtering, and more.

The problem was that we had no company-level contract around how grids should be built.

That meant every dashboard carried repeated implementation cost.

A developer would spend time rebuilding mechanics that already existed elsewhere.

Another developer would miss a property that another screen had.

A formatter would behave slightly differently.

A callback would be wired differently.

State persistence would work in one place and be missing or incomplete in another.

QA then had to test the same behaviours again and again because each grid had drifted just enough to be risky.

This is where frontend work becomes expensive in a way that is hard to see from the outside.

Nobody opens Jira and writes:

> We lost two days because we copied the same grid implementation again.

But the cost is real.

It appears as slower delivery, repeated QA cycles, inconsistent UX, onboarding friction, and defects caused by missed configuration.

The company was not just paying developers to build new business features.

It was paying developers and QA to repeatedly reconstruct the same grid infrastructure.

That is what I wanted to remove.

## The question that changed the solution

I did not start with:

> How do I make AG Grid nicer?

I started with:

> Why are we paying for the same implementation over and over again?

That question changed the architecture.

The goal was not to hide AG Grid completely.

The goal was to create a company-level wrapper that owned the repeated concerns and exposed a clean, typed interface to feature teams.

In other words:

> Product teams should describe the business screen.
>
> The wrapper should handle the grid mechanics.

That meant the wrapper needed to own the boring but important behaviour:

- common grid defaults
- shared column types
- reusable formatters
- event handling
- state persistence
- state restoration
- search integration
- chart configuration
- consistent UX rules
- safe extension points

The result was a reusable Angular wrapper that turned repeated implementation into platform capability.

## Designing the wrapper interface

The first important decision was the public interface.

A wrapper like this succeeds or fails based on its contract.

If the API is too thin, consumers still need to know too much about the underlying grid.

If the API is too restrictive, teams cannot build real business features.

The sweet spot is an interface that standardises company behaviour while leaving room for product-specific needs.

A simplified version looked like this:

```ts
export interface AppGridConfig<TData> {
  gridId: string;
  rowData: TData[];
  columnDefs: AppGridColumnDef<TData>[];

  enableQuickSearch?: boolean;
  enableCharts?: boolean;
  enableGrouping?: boolean;

  persistState?: boolean;
  restoreState?: boolean;

  emptyStateTitle?: string;
  emptyStateDescription?: string;
}
```

The important detail is not the exact shape.

The important detail is what this interface removed.

Consumers no longer needed to remember every default.

They no longer needed to manually wire every common behaviour.

They no longer needed to copy state persistence from a previous screen.

They no longer needed to know which formatters were approved for currency, quantity, dates, or percentages.

They could focus on the business case.

That is the point of a good abstraction.

It does not remove flexibility.

It removes unnecessary decisions.

## From callback soup to an event-driven model

One of the biggest improvements was changing how consumers reacted to grid behaviour.

Before the wrapper, the common approach was to pass callbacks directly through `gridOptions`.

It usually looked something like this:

```ts
const gridOptions: GridOptions<OrderRow> = {
  onCellValueChanged: (event) => {
    recalculateTotals(event.data);
    saveDirtyState();
  },
  onFilterChanged: () => {
    analytics.track('orders-grid-filter-changed');
  },
  onColumnMoved: () => {
    saveLayout();
  }
};
```

For one screen, this is fine.

Across a large dashboard platform, it becomes messy.

Callbacks get scattered across components.

Different teams implement similar behaviour slightly differently.

Cross-cutting concerns like persistence, analytics, telemetry, validation, and dirty-state handling become harder to standardise.

So I changed the model.

Instead of encouraging every consumer to wire raw AG Grid callbacks, the wrapper captured grid events internally and exposed a typed event bus.

Consumers subscribed to meaningful events:

```ts
this.gridEventsBus
  .on(Events.CellValueChanged<OrderRow>())
  .subscribe(({ row, column, value }) => {
    recalculateTotals(row);
    saveDirtyState();
  });
```

Another consumer could listen for filters:

```ts
this.gridEventsBus
  .on(Events.FilterChanged)
  .subscribe(() => {
    analytics.track('orders-grid-filter-changed');
  });
```

And persistence could be handled centrally:

```ts
this.gridEventsBus
  .on(Events.StateUpdated)
  .subscribe(({ state }) => {
    this.gridStateStore.save('orders-grid', state);
  });
```

This was a major architectural improvement.

It decoupled the grid from its consumers.

It gave us a single place to normalise raw grid events.

It allowed multiple parts of the product to react to the same event without turning `gridOptions` into a dumping ground.

It also made the wrapper feel like a real product API, not just a pass-through component.

## Custom column types created consistency by default

The next high-impact improvement was standardising column types.

In a dashboard-heavy product, the same kinds of values appear everywhere:

- prices
- quantities
- dates
- percentages
- statuses
- IDs
- links
- action columns

Before the wrapper, each team could format those slightly differently.

One price column might show two decimals.

Another might show none.

One date column might use local time.

Another might use a backend string directly.

One quantity column might be aligned properly.

Another might not.

These inconsistencies look small, but they create product friction.

They also create QA friction because the tester has to ask:

> Is this difference intentional or just another implementation drift?

So the wrapper introduced custom column types.

```ts
export const APP_COLUMN_TYPES = {
  price: {
    type: ['numericColumn'],
    cellDataType: 'number',
    valueFormatter: ({ value }) => formatCurrency(value),
    filter: 'agNumberColumnFilter',
  },

  quantity: {
    type: ['numericColumn'],
    cellDataType: 'number',
    valueFormatter: ({ value }) => formatQuantity(value),
    filter: 'agNumberColumnFilter',
  },

  date: {
    cellDataType: 'dateString',
    valueFormatter: ({ value }) => formatShortDate(value),
    filter: 'agDateColumnFilter',
  },

  percentage: {
    type: ['numericColumn'],
    cellDataType: 'number',
    valueFormatter: ({ value }) => formatPercentage(value),
  }
} as const;
```

Now a developer did not need to reinvent a price column.

They could declare intent:

```ts
{
  field: 'totalPrice',
  headerName: 'Total Price',
  type: 'price'
}
```

That is a tiny line of code, but a big product improvement.

The correct formatting, filtering, alignment, and display behaviour came from the wrapper.

Fixes happened once.

Improvements happened once.

Consistency became the default.

## Persisting and restoring state

One of the most visible improvements for users was state persistence.

Enterprise users do not just use a dashboard.

They shape it around their workflow.

They resize columns.

They hide columns.

They reorder columns.

They apply filters.

They sort data.

They build a layout that helps them work faster.

Before the wrapper, this behaviour was inconsistent because each grid could implement persistence differently, or not at all.

So persistence became a first-class wrapper responsibility.

Each grid had a stable `gridId`.

When the grid state changed, the wrapper captured it.

When the user returned, the wrapper restored it.

A simplified version:

```ts
const gridOptions: GridOptions<TData> = {
  initialState: this.gridStateStore.get(config.gridId) ?? undefined,
};
```

And when state changed:

```ts
this.gridEventsBus
  .on(Events.StateUpdated)
  .pipe(debounceTime(250))
  .subscribe(({ api }) => {
    this.gridStateStore.save(config.gridId, api.getState());
  });
```

This made the experience feel more polished for users.

But it also mattered architecturally.

State persistence is exactly the sort of feature that becomes expensive when every team implements it separately.

By embedding it in the wrapper, we turned it into a platform feature.

Every dashboard benefited.

## Why this saved real money

I am deliberately not publishing internal company numbers, but the economics were clear.

Before the wrapper, every new grid-heavy feature carried a repeated cost:

- duplicated implementation
- missing configuration
- inconsistent formatting
- repeated QA validation
- bug fixing caused by drift
- extra onboarding time for developers
- slower delivery for product teams

After the wrapper, much of that work disappeared.

The company paid once for the shared platform capability instead of paying repeatedly for each feature team to rebuild the same mechanics.

The rough cost model looked like this:

```text
monthly cost avoided =
  number of grid-heavy deliveries per month
  × repeated developer effort removed per delivery
  × repeated QA effort removed per delivery
  × blended cost of that time
```

When you multiply that across multiple dashboards, developers, QA cycles, and releases, the number grows quickly.

That is how a frontend wrapper can save tens of thousands every month.

Not because it is magical.

Because it removes repeated paid effort.

## What changed after the wrapper

The technical result was cleaner code.

But the business result was more important.

New dashboards became faster to build.

Developers had fewer decisions to make for standard behaviour.

QA had fewer duplicated scenarios to rediscover.

Users got more consistent interactions.

Product teams could ship grid-heavy features with more confidence.

The wrapper also created a better foundation for future improvements.

Once a behaviour lived centrally, we could improve it centrally.

If we improved the date formatter, every relevant grid benefited.

If we improved persistence, every grid benefited.

If we added telemetry, every grid could emit consistent events.

If we fixed an edge case, we fixed it once.

That is the difference between component reuse and platform leverage.

## What recruiters and hiring managers should take from this

The important part of this story is not:

> I built an AG Grid wrapper.

The important part is:

> I identified a repeated business cost, understood the implementation pattern causing it, designed a reusable interface, and turned duplicated work into a shared platform capability.

That demonstrates more than frontend development.

It shows:

- technical ownership
- product thinking
- cost awareness
- system design
- developer experience focus
- ability to reduce QA and delivery friction
- architecture that supports business outcomes

A lot of engineers can build a table.

Fewer engineers can notice that the company is paying to build the same table mechanics over and over again.

Even fewer can fix that in a way that improves engineering speed, QA efficiency, user experience, and monthly cost.

That is the work I am proud of.

## The lesson

The best abstractions are not clever.

They are useful.

They remove repeated cost.

They make the right thing easy.

They reduce the surface area for mistakes.

They turn one team’s solution into everyone’s default.

That wrapper mattered because it changed the economics of delivery.

It turned copy-paste implementation into a reusable company asset.

It reduced toil.

It reduced drift.

It improved consistency.

And it gave the business back time and money every month.

That is the kind of architecture I like building:

not abstraction for abstraction’s sake, but architecture that makes delivery cheaper, faster, and safer.
