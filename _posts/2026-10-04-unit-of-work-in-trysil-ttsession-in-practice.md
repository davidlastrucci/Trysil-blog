---
title: "Unit of Work in Trysil: TTSession<T> in practice"
date: 2026-10-04 08:27:00 +0200
categories: [Tutorials]
tags: [session, unit-of-work, tutorial]
---

Back in *Why Trysil uses full-clone in TTSession*, we argued why `TTSession<T>` clones every entity on entry and drives state with explicit API calls. That post was about the *why*. This one is about the *how it reads in real code*.

The short version: `TTSession<T>` is the Trysil way of saying *"here's a set of entities I'm going to work with; I'll tell you which ones to Insert, Update, or Delete; commit them all in one transaction"*. The session hands you clones of your entities to mutate, and you mark each one with `Insert`, `Update`, or `Delete` as you decide what to do.

## The shape of a session

```pascal
procedure BumpSmallOrders(LContext: TTContext);
var
  LOrders: TTList<TOrder>;
  LSession: TTSession<TOrder>;
  LOrder: TOrder;
begin
  LOrders := LContext.CreateEntityList<TOrder>();
  try
    LContext.Select<TOrder>(LOrders, TTFilter.Empty());

    LSession := LContext.CreateSession<TOrder>(LOrders);
    try
      for LOrder in LSession.Entities do
        if LOrder.Amount < 100 then
        begin
          LOrder.Amount := LOrder.Amount * 1.10;
          LSession.Update(LOrder);
        end;

      LSession.ApplyChanges;
    finally
      LSession.Free;
    end;
  finally
    LOrders.Free;
  end;
end;
```

You select into your own list, hand that list to `CreateSession<T>`, then work on `LSession.Entities` — those are clones of what was in `LOrders`. Every clone you call `Update` on gets written in one transaction at `ApplyChanges`; every clone you don't call `Update` on is left alone.

There is no automatic diff. The session tracks a state per entity — `Original` / `Inserted` / `Updated` / `Deleted` — and the state is driven entirely by your calls. Mutating `LOrder.Amount` without calling `LSession.Update(LOrder)` leaves the entity in `Original` state and nothing is written. Calling `LSession.Update(LOrder)` without mutating any field still writes — the session doesn't second-guess you.

## Three operations the session handles

```pascal
procedure EditAndAdd(LContext: TTContext);
var
  LOrders: TTList<TOrder>;
  LSession: TTSession<TOrder>;
  LNewOrder: TOrder;
begin
  LOrders := LContext.CreateEntityList<TOrder>();
  try
    LContext.Select<TOrder>(LOrders, TTFilter.Empty());

    LSession := LContext.CreateSession<TOrder>(LOrders);
    try
      // 1. Mutate an existing entity
      LSession.Entities[0].Amount := 999;
      LSession.Update(LSession.Entities[0]);

      // 2. Add a new entity
      LNewOrder := LContext.CreateEntity<TOrder>;
      LNewOrder.CustomerID := 42;
      LNewOrder.Amount := 50;
      LSession.Insert(LNewOrder);

      // 3. Remove an entity
      LSession.Delete(LSession.Entities[LSession.Entities.Count - 1]);

      LSession.ApplyChanges;
    finally
      LSession.Free;
    end;
  finally
    LOrders.Free;
  end;
end;
```

`ApplyChanges` walks the session's entities, and each non-`Original` state emits one SQL statement:

- `UPDATE` for the clone marked with `Update`.
- `DELETE` for the clone marked with `Delete`.
- `INSERT` for the entity passed to `Insert`.

All inside one transaction. Either every statement commits or none do.

One line in there hands over ownership, and it is worth stopping on:

```pascal
LNewOrder := LContext.CreateEntity<TOrder>;
LSession.Insert(LNewOrder);
```

From `Insert` on, **the entity belongs to the session**, which frees it when it is destroyed. Do not free it yourself and do not use it after `LSession.Free`. Everywhere else in Trysil the rule is the opposite - without an identity map the caller owns what it created - and `Insert` is where it stops. That is deliberate: a grid posting a new row hands the entity over and has no later moment at which it could free it. With an identity map in play the map owns the entity instead, and the session's list does not.

## Free the session before the context

The order in every example above is not incidental:

```pascal
LSession := LContext.CreateSession<TOrder>(LOrders);
try
  // ...
finally
  LSession.Free;      // the session first
end;
// ... and the context after
```

`CreateSession<T>` hands the session three references it borrows from the context: the connection, the provider and the resolver. None of them tells it when they die, and the session **uses two of them while it is being destroyed**, to release its clones through the same disposal path every other entity goes through - so that a clone's lazy members go with it, and its entry leaves the undo log a rollback would otherwise write into.

An application that keeps the context and the session as fields and frees them in declaration order, context first, reads a destroyed object. Nothing in the type system stops it. The `try..finally` around the block that uses the session is the shape that is always right, and it is the reason every example in this post is written that way.

## Rollback is automatic on exception

If `ApplyChanges` throws — say, one of the inserts fails validation — the transaction rolls back. Every other change is undone. You're back to the database state from before `ApplyChanges` was called.

```pascal
try
  LSession.ApplyChanges;
except
  on E: ETValidationException do
    HandleFormErrors(E.Errors);   // database is untouched
end;
```

The session doesn't flag itself as "permanently failed" — the state dictionary is still intact. Entities you marked `Updated` are still `Updated`, entities you marked `Inserted` are still `Inserted`. Fix what was wrong, call `ApplyChanges` again.

## What the SQL looks like

Each state maps to one SQL statement with predictable shape:

- `Updated` → `UPDATE` rewriting every mapped column of the row (not just the changed ones — the session doesn't know which are changed, and the `UPDATE` command writes the full row either way).
- `Inserted` → `INSERT` with all mapped columns.
- `Deleted` → `DELETE`, or — if the entity has `[TDeletedAt]` — an `UPDATE` that marks the row as soft-deleted.
- `Original` → no SQL.

A few fields that behave specially when an `UPDATE` runs:

- **`[TVersionColumn]`** is written as `col = col + 1` in the generated SQL, and after a successful `UPDATE` the resolver bumps the version on the clone. Any manual assignment you made to the version field before calling `Update` is overwritten.
- **`[TUpdatedAt]` / `[TUpdatedBy]`** are populated by the resolver before the SQL runs — from `Now` and from `OnGetCurrentUser`. Any manual assignment on those fields is overwritten too.
- **`TTNullable<T>`** — null and value are both written as-is, no special handling required of the caller.
- **Primary key** — `Update` uses the PK in the `WHERE` clause. Mutating the PK on a session clone is a misuse: you'd be writing to a row that may not exist.

## The one-transaction guarantee

Under the hood, `ApplyChanges` opens a `TTTransaction` (if one isn't already open on the connection), iterates the session's entities in their state-dictionary order, and calls the resolver's `Insert` / `Update` / `Delete` for each non-`Original` entry. If any call raises, the transaction rolls back and the exception propagates. No partial writes.

## When to use the session vs. direct calls

The session is the right tool when:

- **You want the originals safe until commit.** Edit dialogs with cancel semantics: the user can close the form without saving and nothing in memory has changed.
- **You're reconciling a set of entities and a set of decisions.** Bulk edits across a list where you iterate, decide per row whether it needs `Update` / `Delete`, possibly `Insert` new rows — all committed together or not at all.
- **Transactionality across mixed operations matters.** Inserts, updates, and deletes as one logical step.

Direct `Insert<T>` / `Update<T>` calls are better when:

- One-shot writes. Adding a log row, toggling a flag on a single entity.
- Script-like utility code where cloning is overhead you don't need.
- You want to choose the order of operations yourself — the session iterates in entity order, not in a reordered `inserts → updates → deletes` sequence.

If you find yourself reaching for a session for one-row updates, you're over-engineering.

## Scope: one type per session

`TTSession<T>` is generic over one entity type. If you need to edit orders and customers in the same unit of work, you spawn two sessions. They share the context (and thus the transaction, if you wrap both in one explicit transaction), but each has its own list and its own clones.

```pascal
LOrderSession := LContext.CreateSession<TOrder>(LOrders);
LCustomerSession := LContext.CreateSession<TCustomer>(LCustomers);
try
  // ... edit both ...
  LOrderSession.ApplyChanges;
  LCustomerSession.ApplyChanges;
finally
  LCustomerSession.Free;
  LOrderSession.Free;
end;
```

For strictly-atomic behavior across two entity types, open an explicit `TTTransaction` around both `ApplyChanges` calls. Otherwise, each session opens its own transaction — fine for most UIs, not fine for multi-table invariants.

## Interaction with the identity map

The session works orthogonally to the identity map. Whatever state the map is in, the session clones the originals you pass to `CreateSession<T>` — `LSession.Entities` holds clones, not mapped instances. Mutations on the clones don't touch the originals (or the map) until `ApplyChanges` writes to the database.

The clearest way to use the two together is: select into a list, hand that list to a session, edit clones, apply. Don't mutate the originals in the outer list in parallel — the session doesn't know about those mutations, and with the map on you'd be changing the instance the context has handed out elsewhere.

## What the session is not

- It's not a cache across requests. Its lifetime is its `try/finally`. When you free it, everything is gone.
- It's not a lazy loader. `TTLazy<T>` properties on entities inside a session still fetch on demand; the session doesn't prefetch related entities.
- It's not a query builder. It holds whatever you put into its list — selected or constructed. The session doesn't know how the list was filled.

## Closing

`TTSession<T>` turns a list of entities plus a set of explicit decisions (`Insert` / `Update` / `Delete`) into one transactional apply. The session gives you an isolated workspace of clones and a state machine that's driven entirely by your calls — no hidden diffs, no inferred intent. When you want to skip the workspace and write directly, `Insert<T>` / `Update<T>` / `Delete<T>` on the context do the same job for one-off operations.

Next: `ApplyAll<T>` without a session — passing three ready-made lists (inserts, updates, deletes) to the context when you already know the shape of the batch.
