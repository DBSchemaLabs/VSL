# VSL: Versioned Schema Language

VSL is a proposal for changing how a value is stored in a relational database while the application keeps running, including while old and new versions of it are live at the same time, without rewriting the application's queries.

This is the short version. The full problem statement, with the evidence behind it, is in [Research/Problem_Statement.md](https://github.com/DBSchemaLabs/Research/blob/main/Problem_Statement.md). It sets out nine requirements, reviews our own experiments, eight research papers, industry tools and four industry case studies against them, and ends with the gaps a solution has to close (Section 7). The objectives below follow from those gaps.

## The problem

Some schema changes keep the information the same and change only how it is stored, for example moving a column of status words to small integer codes. Even so, such a change has to happen while the database serves traffic, while old and new versions of the application run side by side during a rolling deploy, and while other writers such as scheduled jobs keep using the same tables.

Today's options each give something up:

* Changing the column in place rewrites the table while holding a lock that stalls other queries.
* Hiding the change behind views and triggers keeps old queries running, but loses indexes, breaks upserts and slows every write.
* Expand/contract by hand works, but puts dual-write code into the application and depends on each team rediscovering the same rules.

Research systems solve parts of the problem: rewriting old queries, keeping several schema versions writable, moving data without blocking, fencing out stale writers. Each solves its part in isolation, and none covers an application in the middle of a rolling deploy.

## What a solution must do

These are the requirements from the problem statement:

1. No migration step makes other queries wait longer than an agreed limit.
2. Old and new versions of the application are served at the same time.
3. Every query keeps its meaning, including pattern matches, ordering, comparisons and parameterised statements.
4. Selective reads keep using their indexes.
5. All writes are accepted, including upserts, and old and new representations stay consistent.
6. A stale or paused client cannot write in a shape the migration has moved past.
7. Writers that bypass the application are kept consistent.
8. The mapping between representations is stated once, in both directions.
9. The application's queries can be checked against every stage before deployment.

## Constraints

* The database stays online throughout. Maintenance windows are not an option.
* Rolling deploys are the norm, so old and new application versions always overlap.
* Not every writer can be changed. Scheduled jobs, ad hoc sessions and other services may write directly to the database.
* Existing ORMs and migration tools stay in place. VSL works alongside them rather than replacing them.
* Mappings that lose information, such as merging two statuses into one, are detected and refused rather than supported.
* The approach should not depend on features of one particular database.

## Objectives

Each objective answers one of the five gaps in the problem statement.

1. Make the mapping a first-class declaration. A change is described once: the old representation, the new one, and the mapping between them in both directions, marked as reversible or lossy. The same declaration drives data movement, query translation, consistency checks and rollback.
2. Translate queries before they reach the database. Each application query is rewritten for the current stage of the migration, so the database only ever sees plain SQL against real tables, using values its indexes can serve. Queries that cannot be translated faithfully are rejected, not silently changed.
3. Make the migration's stage explicit and enforced. A migration moves through known stages (expand, dual-write, backfill, switch reads, contract). The current stage is durable and visible to every client, and the database refuses writes from a client that has fallen behind.
4. Provide a temporary safety net for other writers. While a migration is in progress, the database itself keeps old and new representations consistent for writers that don't go through VSL. Once the migration completes, the safety net is removed.
5. Catch problems before they ship. The application's real queries are checked against every stage of a migration before it runs, and every step carries a limit on how long it may make other queries wait.

## How we will measure it

We will re-run the scenario from our own experiments (a two-million-row table whose text column is re-encoded as small integers, under a live mixed workload) with VSL and with the existing approaches, and check that:

* neither the old nor the new version of the application sees an error at any point;
* pattern matches, ordering, comparisons, upserts and prepared statements behave the same before, during and after the change;
* selective reads keep their index scans;
* write overhead is no worse than hand-written expand/contract;
* no query waits longer than the agreed limit behind any migration step;
* a stale client's write is rejected, never applied;
* storage returns to its original size once the change is complete.

## Status

Problem statement and objectives. Design and prototype to follow.
