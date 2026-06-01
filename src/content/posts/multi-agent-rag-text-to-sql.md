---
title: "I Built My First RAG and Skipped the Vector Database on Purpose"
description: "A technical teardown of a multi-agent, text-to-SQL RAG built on LangGraph for a large ERP. Plain-English questions in, a correctly-shaped AG Grid out. Why I skipped embeddings and a vector store, how the agent team verifies its own SQL, and the row-level governance problem I did not solve."
pubDate: 2026-06-02
tags: ["RAG", "LangGraph", "text-to-SQL", "multi-agent", "AG Grid", "AI", "ERP", "data-governance"]
readingTime: "8 min read"
draft: false
---

My first RAG did not touch a single embedding. No vector store. No cosine similarity. For a long time I assumed "RAG" implied "stuff documents into a vector DB and pray." It does not. Retrieval-Augmented Generation just means the model retrieves grounded facts before it answers. Where those facts live is your call. Mine lived in a relational ERP database, and the people asking questions wanted rows, not paragraphs.

This is a teardown of that proof of concept. I built it at Scopeworker, a large ERP, with one goal: let a customer type a plain-English question and get back an AG Grid laid out the way they wanted. No report builder. No SQL. No "open a ticket with the data team and wait three days."

## Why not a vector database

The default RAG tutorial chunks your PDFs, embeds them, and does nearest-neighbor search at query time. That is the right tool when your source of truth is unstructured prose and "close enough" is a valid answer.

An ERP is the opposite on both counts.

The source of truth is a normalized relational schema. The answer to "show me all open contracts over 50k that renew this quarter" is not a fuzzy semantic match. It is an exact set of rows that either satisfies the predicate or does not. Embeddings smear that precision into a similarity score, which is exactly the property you do not want when a number on the screen drives a purchasing decision.

So the retrieval step here is not vector search. It is SQL. The hard part moves from "find the relevant chunk" to "generate a query that is correct, safe, and returns what the human actually meant." That reframing is the whole post.

## The shape of the system

One orchestrator on top. Underneath, a small team of agents that each do one job and hand off. Each one is dumb in a useful, single-minded way, and that is the point. LangGraph models this well because the unit of work is a node, and the edges are explicit. You can see the loops. You can see the escape hatches.

Here is the flow.

```
                          ┌─────────────────┐
   user question  ──────► │   Orchestrator  │  reads intent, routes
                          └────────┬────────┘
                       small talk  │  data question
                  ┌────────────────┴───────────────┐
                  ▼                                 ▼
         ┌─────────────────┐              ┌───────────────────┐
         │  Conversation   │              │   Database agent  │  (a team)
         │     agent       │              └─────────┬─────────┘
         └─────────────────┘                        │
                                                     ▼
                                          ┌─────────────────────┐
                                          │   Semantic agent    │ human terms → real tables
                                          └──────────┬──────────┘
                                                     ▼
                                          ┌─────────────────────┐
                                          │   SQL generator     │ ◄──────────┐
                                          └──────────┬──────────┘            │
                                                     ▼                       │ regenerate
                                          ┌─────────────────────┐            │
                                          │   SQL verifier      │ ──bad──────┘
                                          └──────────┬──────────┘   still bad → ask human
                                            query looks right
                                                     ▼
                                          ┌─────────────────────┐
                                          │   Execution tool    │ runs approved SQL
                                          └──────────┬──────────┘  (view-only DB user)
                                                     ▼
                                          ┌─────────────────────┐
                                          │   Result verifier   │ rows vs. the question
                                          └──────────┬──────────┘  valid-but-wrong → loop
                                                     ▼
                                          ┌─────────────────────┐
                                          │    Display agent    │ grid / AG Grid / JSON
                                          └─────────────────────┘
```

Instead of one H3 per box, let me walk it as a chain of failures. Each agent exists because something breaks without it.

### Problem 0: the model talks to the database when you just said hello

The orchestrator reads intent and routes. That is the entire job, and keeping it that small matters. If the user says "morning, hope you're well," that goes to the conversation agent and never gets near the database. If they ask a real data question, it goes down the database path. A thin router is easy to reason about and easy to guard. The conversation agent then handles the small talk so the database side never has to pretend to be friendly. Separation of concerns, but for tone.

### Problem 1: the model invents table names, and it does not error, it lies fluently

This is where naive text-to-SQL dies. A customer says "contracts." The table is called `tickets`. Nobody renamed it because ten years of code depends on that name. If you hand the raw question straight to a SQL-writing model, it confidently invents a `contracts` table. The query fails or, worse, silently returns nothing, and the user concludes the AI is useless.

The fix is the semantic agent, the one that earns its keep. It is a translation layer that maps human vocabulary to the real schema before any SQL exists. "Contracts" becomes `tickets`. "Vendor" becomes `supplier_account`. "Renews this quarter" becomes a date predicate against the right column. The model is no longer guessing at structure. It is filling in a query over tables it has been told are real.

### Problem 2: somebody still has to write the join

The SQL generator takes the mapped tables plus the intent and writes the query. Because the semantic step already resolved the names, this agent does the part it is actually good at: composing joins, filters, and aggregates over a known shape.

### Problem 3: the query is wrong and you find out by running it on production

Before a single byte hits the DB, a verifier reads the generated SQL against the original question and asks: does this query actually answer what was asked? If not, it loops back to the generator to try again.

The important word is *loop*, not *infinite loop*. After a bounded number of regenerations, if the query still does not hold up, the system stops guessing and asks the human for more context. A loop with a door, not a hamster wheel. Retrying forever is how you turn a bug into a bill.

### Problem 4: the query is valid, runs clean, and still returns the wrong rows

The execution tool runs the approved SQL. Nothing clever here on purpose. The cleverness is in the credentials it runs under, which I cover in guardrails.

Then comes the subtle one. A query can be syntactically perfect, execute cleanly, and return the wrong answer. Someone asks for "this quarter" and the predicate captured last quarter. The rows are real. They are just not the rows that were requested. So after execution, a result verifier compares what came back against what the user asked for. If the data does not match the intent, that counts as a failure and the system loops. Most text-to-SQL demos stop at "the query ran." Production starts at "the query ran and the answer is right."

### Problem 5: the agents that fetch data start deciding pixels

The display agent's only job is *how* to show the result. Same data, three presentations:

- a small inline grid for a handful of rows,
- a full AG Grid with proper column definitions for anything real,
- raw JSON when the consumer is another system.

Splitting this out matters. The agents that fetch data should not also be deciding layout. When the display agent picks the full AG Grid path, it produces `columnDefs` with formatters and getters, like this:

```typescript
import type { ColDef, ValueGetterParams, ValueFormatterParams } from "ag-grid-community";

interface ContractRow {
  ticketId: string;
  supplierName: string;
  amount: number;       // minor units (cents)
  currency: string;
  renewsOn: string;     // ISO date
}

const currencyFmt = (p: ValueFormatterParams<ContractRow, number>) =>
  p.value == null
    ? ""
    : new Intl.NumberFormat("en-US", {
        style: "currency",
        currency: p.data?.currency ?? "USD",
      }).format(p.value / 100);

export const contractColumnDefs: ColDef<ContractRow>[] = [
  { field: "ticketId", headerName: "Contract", pinned: "left", width: 140 },
  { field: "supplierName", headerName: "Vendor", flex: 1, filter: true },
  {
    field: "amount",
    headerName: "Value",
    type: "rightAligned",
    valueFormatter: currencyFmt,
    aggFunc: "sum",
  },
  {
    headerName: "Renews",
    // valueGetter so sorting uses a Date, not a string
    valueGetter: (p: ValueGetterParams<ContractRow>) =>
      p.data ? new Date(p.data.renewsOn) : null,
    valueFormatter: (p) =>
      p.value ? (p.value as Date).toLocaleDateString("en-GB") : "",
    sort: "asc",
  },
];
```

Note the `valueGetter` returning a `Date` so the grid sorts chronologically instead of alphabetically, and the `valueFormatter` doing currency math on the minor units. That is the layout the customer "wanted" without ever knowing the word `columnDefs` exists. AG Grid's [column definition docs](https://www.ag-grid.com/javascript-data-grid/column-definitions/) go deep on the rest.

## A LangGraph node sketch

To make the wiring concrete, here is the database subgraph in LangGraph-flavored TypeScript. Trimmed, but the structure is real: nodes mutate shared state, and a conditional edge decides whether to loop or move on.

```typescript
import { StateGraph, START, END } from "@langchain/langgraph";

interface DbState {
  question: string;
  mappedTables?: string[];
  sql?: string;
  sqlOk?: boolean;
  attempts: number;
  rows?: unknown[];
  resultOk?: boolean;
}

const MAX_ATTEMPTS = 3;

const graph = new StateGraph<DbState>({ channels: /* ... */ {} })
  .addNode("semantic", semanticMapNode)     // human terms -> real tables
  .addNode("generate", sqlGeneratorNode)    // intent + tables -> SQL
  .addNode("verifySql", sqlVerifierNode)    // does the SQL match the ask?
  .addNode("execute", executeNode)          // view-only user runs it
  .addNode("verifyResult", resultVerifierNode)
  .addNode("askHuman", clarifyNode)         // the escape hatch
  .addNode("display", displayNode);

graph.addEdge(START, "semantic");
graph.addEdge("semantic", "generate");
graph.addEdge("generate", "verifySql");

// loop with a bounded escape hatch, not an infinite retry
graph.addConditionalEdges("verifySql", (s: DbState) => {
  if (s.sqlOk) return "execute";
  if (s.attempts >= MAX_ATTEMPTS) return "askHuman";
  return "generate";
});

graph.addEdge("execute", "verifyResult");
graph.addConditionalEdges("verifyResult", (s: DbState) =>
  s.resultOk ? "display" : (s.attempts >= MAX_ATTEMPTS ? "askHuman" : "generate")
);

graph.addEdge("display", END);
graph.addEdge("askHuman", END);
```

If you want the canonical patterns, the [LangGraph docs](https://langchain-ai.github.io/langgraph/) cover state channels and conditional edges properly. The shape above is the part worth stealing: every loop has a counter and an exit.

## The guardrails, boring on purpose

Good safety is dull. Clever is fragile. Boring is what you want around a database. Three boring things did most of the work.

**A dedicated database user.** View-only permissions, scoped to a handful of curated tables. The agent literally cannot write. It cannot even *see* most of the schema. The most reliable way to stop an agent from dropping a table is to make "drop" a permission it was never granted. If the model has a bad day and tries to drop a table, the database laughs at it. Prompts are suggestions. Grants are walls. Security you can prove beats security you hope for.

**Enriched schemas.** I hand-augmented the curated table definitions with descriptions and relationships. Instead of the model reverse-engineering what `tickets.status_code = 4` means, it was told. Less guessing, more grounding. This is the actual "retrieval" in this RAG: pulling trustworthy schema context into the prompt. The least exciting work did the most.

**Topic blocks.** Hard guardrails so nobody could steer the thing into chatting about the weather or, my favorite test case, pancakes. It is a data assistant. It stays a data assistant. If you ask it for a pancake recipe, it declines, and I sleep fine.

## The problem I did not solve

Here is the honest part, because a teardown that only shows the clean bits is marketing.

Data governance. At Scopeworker, row-level access control lived in application **code**, not in the database. The service layer decided which records a given user was allowed to see. The view-only database user I gave the agent enforced safety at the *table* level and knew nothing about *record*-level permissions. Table-level walls do not help when the rule you need lives one floor up.

Sit with that. A generated query, perfectly valid and running under a perfectly locked-down read-only account, could in principle return rows that a specific user should never have been shown. The table guardrail was real. The row guardrail was not. The SQL was right. The governance was wrong. That keeps me up more than any prompt injection. For a proof of concept on curated, low-sensitivity tables, acceptable. For general rollout, a non-starter.

The roadmap fix is to stop letting the agent write raw SQL for anything sensitive. Route those requests through the existing authorized backend endpoints that already enforce RBAC. The agent decides *what* to ask. The trusted backend decides *what it is allowed to return*. The agent gets demoted from "writes SQL against the database" to "calls the same governed API a human user would have called," and the years of access-control logic already baked into that API come along for free.

## What I would keep

- **Skip the vector store when the truth is relational.** RAG is retrieval, not embeddings. Match the retrieval mechanism to where the facts actually live.
- **A semantic mapping layer is non-negotiable.** Most text-to-SQL failures are not bad SQL. They are the model hallucinating table names. Resolve vocabulary to schema *before* you generate.
- **Verify twice: the query and the result.** Valid-but-wrong is a failure mode, and it is the one that quietly erodes trust.
- **Every loop needs a counter and an exit.** Ask the human before you retry forever.
- **Put safety in permissions, not prompts.** A read-only user on curated tables beats any amount of "you must not" instruction text.
- **Separate fetching from formatting.** The display agent earning its own box is why the same rows can become an inline grid, a full AG Grid, or JSON without the data agents caring.

The architecture I am confident about. The governance gap is the part I am still chewing on, and it generalizes well beyond my ERP. The moment an agent can author its own queries, your row-level access model has to live somewhere the agent cannot route around.

So I will end where the real conversation starts: how are you handling row-level governance when an agent writes the queries?
