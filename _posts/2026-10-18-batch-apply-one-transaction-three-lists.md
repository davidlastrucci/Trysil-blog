---
title: "Batch apply: one transaction, three lists"
date: 2026-10-18 08:49:00 +0200
categories: [Tutorials]
tags: [batch, transactions, tutorial]
---

Last post looked at `TTSession<T>`, the unit-of-work layer: you hand it a list, work on clones, and mark each one with `Insert`, `Update` or `Delete`. This post is about the other way to write a batch: `ApplyAll<T>`, which takes three lists you have already partitioned and writes them in one transaction.

The session is the right tool when the decisions arrive one at a time, from a user editing a grid. `ApplyAll` is the right tool when you have already partitioned the work yourself.

## The signature

```pascal
procedure ApplyAll<T: class>(
  const AInsertList: TTList<T>;
  const AUpdateList: TTList<T>;
  const ADeleteList: TTList<T>);
```

Three lists, all three required: pass an empty list for the work you do not have, never `nil`, because each list is walked without a check. They are processed in order - inserts first, then updates, then deletes - inside a single transaction. If any step fails, the transaction rolls back.

## A typical use case

An import routine reading a CSV and reconciling it with the current database state. You've computed three things:

- Rows present in the CSV but not in the database -> insert
- Rows present in both, with different values -> update
- Rows present in the database but not in the CSV -> delete

You want the three to happen atomically. Either the import succeeded end-to-end, or the database is untouched.

```pascal
procedure ApplyImport(
  const AContext: TTContext;
  const AInsertList: TTList<TProduct>;
  const AUpdateList: TTList<TProduct>;
  const ADeleteList: TTList<TProduct>);
begin
  AContext.ApplyAll<TProduct>(AInsertList, AUpdateList, ADeleteList);
end;
```

One call. One transaction.

## Why the order matters

`ApplyAll` runs inserts first, then updates, then deletes. This order is deliberate.

Inserts first because subsequent updates or deletes might need the new IDs. If an update affects a row that was just inserted in the same batch, running the insert first means the row exists when the update touches it.

Deletes last for the symmetric reason: a delete in the batch shouldn't pull the rug out from under an update or insert that depends on the row still existing during its processing.

This works for *most* cases. It can fail with cyclic foreign keys - say, a self-referential hierarchy where you're deleting a parent whose foreign keys point at a child you're also inserting. For those, you can call `ApplyAll` twice with different groupings, or open the transaction yourself and place the calls in an order you dictate:

```pascal
LContext.RunInTransaction(
  procedure
  begin
    LContext.InsertAll<TProduct>(LFirstBatch);
    LContext.DeleteAll<TProduct>(LOrphans);
    LContext.InsertAll<TProduct>(LSecondBatch);
  end);
```

`RunInTransaction` commits on a clean exit, rolls back and re-raises on an
exception, and joins the transaction already open if there is one. If you would
rather hold the transaction object, ask for the mode explicitly:

```pascal
LTransaction := LContext.CreateTransaction(
  TTTransactionMode.RollbackOnDestroy);
try
  // ...
  LTransaction.Commit;
finally
  LTransaction.Free;
end;
```

`RollbackOnDestroy` is what makes that `try..finally` correct: any path that
does not reach `Commit` rolls back. `CreateTransaction` with no argument still
compiles and means `CommitOnDestroy`, the older behaviour where destruction is
what commits - it is deprecated for the reason you would expect.

`ApplyAll` is the common-case convenience. The manual transaction is the escape hatch.

## Contrast with the session

Given `ApplyAll`, what does a session add?

The session gives you a **workspace**. It clones the rows you loaded, hands you the clones to mutate, and remembers the state you assigned to each one. It does not compute a diff: there is no such thing in Trysil, and the previous post is explicit about it - the state comes from your `Insert` / `Update` / `Delete` calls, not from comparing fields. What the session buys you is the isolation, not the detection.

`ApplyAll` skips the workspace. No clones, no per-entity state, no list for the framework to walk twice: you pass three lists and they are written. Cheaper, and it means the partitioning is yours to get right.

The usual flow:

- CSV import / API webhook with explicit change intent -> `ApplyAll` directly.
- User-facing edit form -> `TTSession<T>`, because you want the isolated clones and one place that knows what the user decided.
- Backend job that computes all three lists and commits -> `ApplyAll`.
- One-off write -> neither; use `Insert<T>` or `Update<T>` or `Delete<T>` directly.

## Transaction semantics

A few details worth knowing:

**`ApplyAll` joins the transaction that is already open, and opens one only if there is none.** It is `RunInTransaction` underneath, and that is the rule everywhere in Trysil: if the connection is already in a transaction the three lists are written inside it, and the commit belongs to whoever opened it. So wrapping two `ApplyAll` calls on two entity types in one `RunInTransaction` gives you one atomic unit, and no nesting happens - which matters, because Trysil does not use savepoints and a genuinely nested transaction raises rather than pretending.

**No transaction, no batch.** On a connection that does not support transactions, `ApplyAll` and the `...All` methods raise `ETException` instead of writing the rows without one.

**The update mode (`KeyAndVersionColumn` vs. `KeyOnly`) from the connection applies.** If you set the mode to `KeyOnly` (optimistic-locking post), `ApplyAll` respects it and omits the version column from the WHERE clause. If you've used that escape hatch, you're explicitly accepting the tradeoff.

**Events fire per row.** `[TBeforeInsertEvent]`, `[TAfterInsertEvent]`, and so on run for each row individually, in the same transaction. If row 7 of 100 inserts fails its validation, rows 1-6 roll back along with row 7, and the remaining 93 rows never run.

**Soft delete still applies.** If the entity has `[TDeletedAt]`, `ApplyAll`'s delete list becomes a batch of `UPDATE`s (one per row) rather than SQL `DELETE`s. The semantics are otherwise identical.

## The granular cousins

`ApplyAll` is the three-list entry point. If you only have one list, Trysil has single-purpose overloads that wrap the same transactional semantics:

```pascal
LContext.InsertAll<TProduct>(LProducts);
LContext.UpdateAll<TProduct>(LProducts);
LContext.DeleteAll<TProduct>(LProducts);
```

Each of them joins the open transaction if there is one, and opens its own otherwise - the same rule as `ApplyAll`. So two `...All` calls are two transactions when you call them bare, and one when you put them inside a `RunInTransaction`.

## An anti-pattern

A tempting shape to avoid:

```pascal
for LProduct in LProducts do
  LContext.Insert<TProduct>(LProduct);
```

That's a thousand single-row transactions for a thousand products. Each commit round-trip is network + fsync + whatever overhead the database has. `ApplyAll` or `InsertAll` over the same list is one transaction - orders of magnitude faster, and atomic.

If you're iterating because each entity needs per-row work that `ApplyAll` can't do (a `[TBeforeInsertEvent]` hook is the usual escape), remember: hooks run inside `ApplyAll` too. You almost never need the explicit loop. When you do need it, at least wrap the whole thing in `RunInTransaction` so there's one commit for the batch.

## A performance note

`ApplyAll` does not switch to `INSERT ... VALUES (...), (...), (...)` multi-row batching or `COPY`/`BULK INSERT`. Each row is a separate statement. The win is transactional, not protocol-level.

For genuine bulk loading (tens of thousands of rows), you're better off dropping to FireDAC directly with the database's native bulk-insert mechanism, then letting Trysil take over for the incremental work. Trysil is an application ORM; it's not trying to replace the database's ETL tooling.

## Closing

`ApplyAll<T>` is the explicit, imperative side of Trysil's write API: three lists, one transaction, inserts-updates-deletes in that order. The session writes its marked entities one by one, in the order of its list; `ApplyAll` is for when you've already partitioned your changes yourself. For everything else - single writes, or work where partitioning would be busywork - stick with the granular calls or let a session figure it out.

Next: `Trysil.JSon`, serializing entities without writing a hand-rolled marshalling layer.
