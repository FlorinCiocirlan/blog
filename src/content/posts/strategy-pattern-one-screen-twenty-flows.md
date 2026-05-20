---
title: "One Screen, Twenty Flows: Taming Variation with the Strategy Pattern"
description: "How a single Angular component serves twenty subtly different workflows without growing a single if-else branch — and why a busy restaurant POS works the same way."
pubDate: 2026-05-20
tags: ["angular", "design-patterns", "architecture", "typescript"]
draft: false
---

A screen that does almost the same thing in twenty subtly different ways is the kind of feature that quietly rots a codebase. You start with two cases and a tidy `if`. A quarter later you have nine cases, three nested ternaries, and a bug that only reproduces on Tuesdays when the user arrives from a particular dashboard tile. The screen still works. Nobody wants to touch it.

I will show the pattern through an analogy I have grown fond of: a point-of-sale terminal in a restaurant that takes orders from many channels at once. The buttons on the screen never change. What happens behind each button changes a lot. Twenty entry points. Six verbs. One screen.

The wrong answer is to put the twenty entry points inside the screen. The right answer is to put them behind the screen.

## The pattern

Define the verbs once, as an interface:

```ts
export interface OrderChannel<T extends OrderContext> {
  loadAvailableItems(ctx: T, section?: MenuSection): Observable<Response>;
  fireToKitchen(params: TicketParams<T>): Observable<Response>;
  scheduleFire(params: TicketParams<T>): Observable<Response>;
  sendToBar(params: TicketParams<T>): Observable<Response>;
  voidTicket(params: TicketParams<T>): Observable<Response>;
  closeTicket(params: TicketParams<T>): Observable<Response>;
}
```

Implement that interface once per channel. Map the channel identifier to the implementation:

```ts
export const channelMap = new Map<Channel, Type<OrderChannel<any>>>([
  [Channel.DineIn,          DineInChannel],
  [Channel.WalkUpTakeout,   TakeoutChannel],
  [Channel.PhoneOrder,      TakeoutChannel],
  [Channel.DeliveryPlatformA, PlatformAChannel],
  [Channel.DeliveryPlatformB, PlatformBChannel],
  [Channel.Catering,        CateringChannel],
  [Channel.CorporateLunch,  CorporateChannel],
  [Channel.BarTab,          BarTabChannel],
  // …
]);
```

Resolve the right one at the component's edge, using whatever your DI container gives you. In Angular that is a factory and a token:

```ts
{
  provide: orderChannelToken,
  useFactory: (route: ActivatedRoute, factory: OrderChannelFactory) =>
    factory.get(route.snapshot.data.session.channel),
  deps: [ActivatedRoute, OrderChannelFactory],
}
```

The component asks for `OrderChannel<T>`. It never asks which one. The template calls `facade.fireToKitchen(...)` and lets the right implementation run.

The win is structural. The component holds no flow knowledge. Adding a twenty-first flow means writing one service and adding one row to the map. The diff fits on a screen and touches nothing the old flows depend on. You can read any single flow end to end without the others in your peripheral vision.

## The restaurant in your phone

A friend of mine runs a small restaurant that, against the laws of physics and labor, takes orders through six channels. Dine-in, walk-up takeout, phone orders, a delivery tablet from one platform, a delivery tablet from a second platform, and catering. Behind the counter sits one point-of-sale terminal. Every channel uses the same six buttons: open a ticket, add an item, fire to the kitchen, modify, close, void.

The first POS he bought tried to model every channel inside the screen. The cashier picked the channel from a dropdown and the buttons changed meaning. "Fire to kitchen" sent a chit on dine-in; it sent a chit *and* started a delivery countdown on platform one; it sent a chit, started a countdown, *and* printed a packing label on platform two; on catering it scheduled the fire for a future time. The dropdown was three lines of code. The conditionals behind the buttons were three thousand.

When a kitchen runner asked for a small change to catering — fire the cold items first, hot items twenty minutes later — the contractor came back with a quote that included rewriting the dine-in flow. Because, of course, it had to.

The second POS he bought looked the same to the cashier and looked nothing like the first one underneath. The six buttons spoke to an abstract "service profile." Each channel was a profile: a small class with the same six methods. Dine-in's `fire` printed a chit. Platform one's `fire` printed a chit and posted a webhook. Catering's `fire` scheduled. The screen did not know.

Adding a seventh channel — a corporate lunch contract with its own billing rules — took an afternoon. They wrote one profile, added one row to the channel map, and shipped. The dine-in flow was not opened, not tested, not at risk. The cashier noticed nothing.

That is the trade the Strategy pattern offers: a small amount of plumbing at the boundary in exchange for arbitrary growth in variation without arbitrary growth in risk.

## When it is worth the plumbing

It is not always worth it. Two flows and an `if` are fine. Three flows and a `switch` are fine. The pattern earns its keep when three things are true at once:

1. **The verbs are stable.** If `fireToKitchen` means something fundamentally different per channel — not just a different fulfillment path, but a different *concept* — you do not have a strategy. You have several different features wearing one uniform. Split them.
2. **The variation is in the behavior, not in the screen.** The pattern hides *how*, not *what*. If each flow needs different fields on the page, you also need composition in the view.
3. **The set of variants will grow.** If you know there will only ever be two, write the `if`. The pattern's value is in absorbing the eleventh and twelfth cases without a rewrite.

When those three line up, the math is unforgiving in favor of the pattern. Without it, every new flow pays a tax on every old flow — a regression risk, a re-test, a cautious reviewer asking "did this touch dine-in?" With it, each new flow is a leaf you can graft on without disturbing the trunk.

## What you give up

Indirection. A new reader cannot follow `fireToKitchen(...)` straight from the click handler to the HTTP call. They land at the interface and have to ask the container which implementation is wired in. That cost is real and worth naming. The mitigation is a single, obvious file — the map — where every flow points at its strategy. If a new engineer can find that file in under a minute, the indirection pays for itself the first time someone adds a flow without breaking the other nineteen.

The screen does not know which flow is running. That is the whole point. It is also, after the third time you ship a new flow on a Friday afternoon without a single regression, a small daily relief.
