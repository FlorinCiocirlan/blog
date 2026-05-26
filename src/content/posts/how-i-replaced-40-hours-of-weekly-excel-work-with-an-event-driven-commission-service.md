---
title: "How I Replaced 40 Hours of Weekly Excel Work With an Event-Driven Commission Service"
description: "Killing a single-person Excel pipeline with a NestJS event-driven microservice, formula-as-code, PDF report generation, and a dead-letter queue for resilience."
pubDate: 2026-05-27
tags: ["nestjs", "microservices", "event-driven", "queues", "automation", "backend"]
readingTime: "7 min read"
draft: false
---

40 hours a week.

That's how long one person in our company spent every month, hand-building commission reports in Excel.

He pulled slot machine data from a distributed database. Aggregated client info from spreadsheets on his laptop. Applied formulas only he understood, buried in files only he could open. Then he exported it through Power BI and sent the result to clients. Sometimes wrong. Always ugly.

The whole department revolved around him. If he quit, we lost the formulas. If he got sick, clients didn't get paid.

This is what I built to replace him.

## The Problem Was Never the Math

The math itself wasn't complicated. Slot machine revenue, applied commission percentages by client tier, deductions for cost centers, totals per period. You could explain the formula on a whiteboard in five minutes.

The problem was everything around the math:

- **Key-person dependency.** The formulas lived in one person's head and in files only he opened. No one else could run a report or verify a number.
- **Manual data assembly.** Client lists changed every month. Cost centers shifted. Someone had to manually reconcile the new state with the old.
- **Unprofessional delivery.** Clients received Excel files exported from Power BI. Formatting drifted. Errors slipped through. No audit trail.
- **No retry mechanism.** If something failed during the month-end push, the only recovery was "redo it by hand."

This isn't a math problem. It's a systems problem.

## The Architecture

The client data already lived in our system. Persisted in our primary database. Updated daily as clients onboarded and cost centers shifted. That was the leverage point — I didn't have to rebuild the data pipeline. I just had to use what was already there.

Three pieces:

1. A **commission service** that owned the formula logic as code.
2. An **event-driven worker** that processed clients off a queue, one at a time.
3. A **dead-letter queue** for jobs that failed past their retry budget.

```text
┌─────────────────┐      ┌──────────┐      ┌──────────────────┐
│ Trigger         │ ───▶ │ Queue    │ ───▶ │ Worker           │
│ (cron, manual)  │      │          │      │ - fetch data     │
└─────────────────┘      └──────────┘      │ - apply formulas │
                              │            │ - generate PDF   │
                              │            │ - deliver        │
                              ▼            └──────────────────┘
                         ┌──────────┐              │ on failure
                         │ DLQ      │ ◀────────────┘
                         │ + retry  │
                         └──────────┘
```

The trigger publishes one message per client. The worker picks them up independently. The database never sees a single transaction trying to compute reports for hundreds of clients at once — it sees one client's worth of queries at a time, spread across whatever throughput the worker pool allows.

## Formula as Code, Not as Knowledge

The first thing I did was extract the formulas out of his head and into the codebase. Every commission rule became a typed, tested function. The business owned them now, not him.

```typescript
@Injectable()
export class CommissionFormulaService {
  computeForClient(
    client: Client,
    period: ReportingPeriod,
    machineRevenue: MachineRevenue[],
  ): CommissionBreakdown {
    const grossByMachine = machineRevenue.map((m) =>
      this.applyMachineTierRate(client.tier, m),
    );

    const gross = grossByMachine.reduce((sum, line) => sum + line.amount, 0);
    const deductions = this.computeCostCenterDeductions(client, period);
    const net = gross - deductions.total;

    return {
      clientId: client.id,
      period,
      grossByMachine,
      deductions,
      net,
      computedAt: new Date().toISOString(),
    };
  }

  private applyMachineTierRate(tier: ClientTier, machine: MachineRevenue) {
    const rate = COMMISSION_RATES[tier][machine.category];
    return { machineId: machine.id, amount: machine.revenue * rate };
  }
}
```

Every rate, every tier, every deduction lives in a config file alongside the code. When the business changes a rate, it's a pull request — reviewed, versioned, deployable, rollback-able.

The day this shipped, the formulas stopped being one person's secret.

## The Real Hard Part: Load

Generating a single client's PDF is fast. Generating PDFs for every client at month-end in a single request would have killed the database.

So I made it event-driven. The trigger fires once at the end of the period and publishes a message per client. Each message is small — just `{ clientId, period }`. The worker fetches, computes, and renders one client at a time.

```typescript
@Controller()
export class CommissionWorker {
  constructor(
    private readonly clients: ClientService,
    private readonly machines: MachineRevenueService,
    private readonly formulas: CommissionFormulaService,
    private readonly pdf: ReportPdfService,
    private readonly delivery: ReportDeliveryService,
  ) {}

  @MessagePattern('commission.generate')
  async handle(@Payload() msg: { clientId: string; period: ReportingPeriod }) {
    const client = await this.clients.findById(msg.clientId);
    const revenue = await this.machines.findForPeriod(client.id, msg.period);

    const breakdown = this.formulas.computeForClient(client, msg.period, revenue);
    const pdfBuffer = await this.pdf.render(client, breakdown);

    await this.delivery.send(client, pdfBuffer, msg.period);
  }
}
```

The database load is now bounded by the worker concurrency. Want to slow things down to protect the cluster? Reduce concurrency. Want it faster? Scale workers horizontally. The architecture gives you a dial instead of a cliff.

## The Dead Letter Queue Is the Safety Net

Failures happen. A client's data is briefly inconsistent. The PDF service times out. A network blip eats a message.

The queue retries each failed message with backoff. If a message still fails after its retry budget, it lands in the dead letter queue. That's where humans come in — not to do the work, but to investigate the edge cases.

```typescript
@MessagePattern('commission.dlq')
async handleDeadLetter(@Payload() msg: FailedCommissionJob) {
  await this.alerts.notifyOps({
    clientId: msg.clientId,
    period: msg.period,
    lastError: msg.error,
    attempts: msg.attempts,
  });

  await this.failedJobs.persist(msg);
}
```

The DLQ does two things: it alerts the on-call engineer, and it persists the failed job so it can be inspected and replayed once the underlying issue is fixed. The system never silently drops a client.

The pattern matters more than the specifics. Retry the transient stuff automatically. Surface the persistent stuff to a human. Never lose the message.

## The Results

| Metric | Before | After |
|---|---|---|
| Manual work per month | ~160 hours | ~0 hours |
| Time per report | hours, by hand | seconds, automated |
| Formula ownership | one person | the codebase |
| Errors per cycle | several, undetected | zero in production |
| Delivery format | Excel from Power BI | Branded PDF |
| Recovery from failure | redo by hand | automatic retry + DLQ |

The department didn't need a head count for this work anymore. The institutional knowledge moved out of one person's laptop and into source control, where every change is reviewed, versioned, and reversible.

Clients started receiving consistent, professional reports on the same day, every month. No more apologetic "let me check that number" emails.

## What Actually Made This Work

A few patterns generalize beyond commission reports.

**Use the data that's already there.** The biggest leverage point was that client data already lived in the system. I didn't have to integrate anything new. If you're automating an Excel workflow, look at what data your application already has — the human is probably doing manual reconciliation that your database already knows about.

**One message per unit of work.** Don't build a "generate all reports" job that processes hundreds of clients in one transaction. Build a "generate one report" worker, then publish one message per client. The infrastructure handles the rest.

**Formulas as code, configs as PRs.** Anywhere business logic lives in someone's head, you're one resignation away from a crisis. Get it into the codebase, even if the initial code is ugly. Refactor later.

**Make failure boring.** Retries handle the transient. The DLQ handles the persistent. An on-call engineer should be reading a structured alert, not a confused email from a finance team.

## Where to Start

If you have an Excel-driven process anywhere in your company, walk through this checklist:

1. Identify the one person who runs it. Ask them to walk you through the workflow end-to-end.
2. Map every input source. Most of them are already in your database.
3. Write the formulas as typed code. Get them reviewed.
4. Pick a queue and a worker pattern. Process one unit of work per message.
5. Set up the dead-letter queue before you ship. You will need it.
6. Run the new system in parallel with the old one for one cycle. Compare outputs.
7. Cut over.

The Excel guy in your company isn't lazy or incompetent. He's just doing what no one else built a system for. Build the system.
