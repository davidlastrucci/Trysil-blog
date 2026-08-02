---
title: "Optimistic locking with TTVersion"
date: 2026-08-02 08:19:00 +0200
categories: [Design Notes]
tags: [concurrency, version-column, internals]
---

Two users open the same invoice. Alice edits the amount and clicks Save. Bob, a minute later, edits the customer and clicks Save. Alice's amount is gone. Bob overwrote it with the value he loaded a minute ago.

This is not a bug in Bob's code. It's the default behavior of every ORM ever written, unless you tell it not to do that. The fix — optimistic locking — is older than most frameworks using it. Trysil wires it in with one attribute and a type.

## The mechanism in one paragraph

You add a column called (conventionally) `VersionID`. Every row has one. Every time Trysil runs an `UPDATE`, it includes the version in the `WHERE` clause *and* increments it:

```sql
UPDATE Invoices
   SET Amount = :amount,
       VersionID = VersionID + 1
 WHERE ID = :id
   AND VersionID = :version
```

If Alice loaded version 7 and Bob loaded version 7, whoever commits first succeeds and bumps the row to 8. The second commit's `WHERE` doesn't match — zero rows affected — and Trysil raises `ETException`. The second user's UI catches the exception, tells them someone else changed the record, and invites them to re-load.

The ORM doesn't lock anything on the database side. The *user* decides, after seeing the conflict, what to do. That's why it's called *optimistic*.

## Wiring it up

```pascal
[TTable('Invoices')]
TInvoice = class
strict private
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TVersionColumn]
  [TColumn('VersionID')]
  FVersion: TTVersion;

  [TColumn('Amount')]
  FAmount: Double;

  [TColumn('CustomerID')]
  FCustomerID: TTPrimaryKey;
public
  property ID: TTPrimaryKey read FID;
  property Version: TTVersion read FVersion;
  property Amount: Double read FAmount write FAmount;
  property CustomerID: TTPrimaryKey read FCustomerID write FCustomerID;
end;
```

Two things:

- **`[TVersionColumn]`** marks the field as the version.
- **`TTVersion`** is the type Trysil uses for version values — internally an `Int32`. You could use a plain `Integer` and Trysil wouldn't complain, but `TTVersion` makes the intent explicit for anyone reading the code.

Both the `TVersionColumn` attribute and the version field's setter are read-only from user code's point of view. You never assign a version manually; the framework manages it.

## The SQL Trysil emits

Three write paths, all affected:

**Insert.** The `VersionID` starts at 1. The application's `CreateEntity<TInvoice>` returns an entity with `VersionID = 0`; Trysil bumps it to 1 before writing the first time. After the insert, the entity in memory reflects version 1.

```sql
INSERT INTO Invoices (Amount, CustomerID, VersionID) VALUES (?, ?, 1)
```

**Update.** The version goes into both `SET` (incremented) and `WHERE` (compared):

```sql
UPDATE Invoices
   SET Amount    = ?,
       CustomerID = ?,
       VersionID = VersionID + 1
 WHERE ID        = ?
   AND VersionID = ?
```

Trysil checks the affected row count after executing. If it's zero, the `WHERE` didn't match — which means either the row is gone (soft-deleted elsewhere, or hard-deleted) or the version moved. Either way, your entity is stale. Trysil raises.

**Delete.** Same pattern:

```sql
DELETE FROM Invoices WHERE ID = ? AND VersionID = ?
```

Zero rows affected → raise. If the entity has `[TDeletedAt]`, the delete becomes the soft-delete UPDATE from the change-tracking post, and that UPDATE also carries the version predicate.

## Reading the version back

After a successful `Insert`, `Update`, or `Delete`, the in-memory entity's version field is refreshed to the new value. This matters: a UI that holds the entity across two edits must see the latest version, or the second save will fail against the ORM itself (which stored the wrong expected version). Trysil handles the refresh inside the resolver — you never touch the field.

## Handling the conflict

In most real codebases, the conflict is caught at the service or controller layer:

```pascal
try
  LContext.Update<TInvoice>(LInvoice);
except
  on E: ETException do
  begin
    if IsOptimisticLockConflict(E) then
      RaiseBusinessError('Someone else changed this record. Please reload.')
    else
      raise;
  end;
end;
```

(There's no separate exception class for the lock conflict; `ETException` with the row-count diagnostic is what you get. A dedicated subclass would be a reasonable enhancement — not there today.)

The HTTP controllers in `Trysil.Http` turn the exception into an HTTP 409 Conflict automatically. We'll see that when we get to the HTTP series.

## Opting out: TTUpdateMode

Some tables don't have a `VersionID`. Log tables, insert-only history tables, data-warehouse rollups. For those, the version column is noise — and wanting to update a row that *doesn't have* a version column without tripping over Trysil's expectations needs a signal.

That signal is `TTUpdateMode` on the connection:

```pascal
LConnection.UpdateMode := TTUpdateMode.KeyOnly;
```

Two values:

- **`KeyAndVersionColumn`** (default). Every `UPDATE` and `DELETE` includes both the primary key and the version column in the `WHERE`. Requires `[TVersionColumn]` on the entity.
- **`KeyOnly`**. `UPDATE` and `DELETE` use only the primary key. No optimistic locking. Use when the table genuinely can't have a version column — bulk imports, replicated staging tables — or when the surrounding application has its own locking strategy.

The setting is per-connection, not per-entity. If you need it per-entity, use two connections with different update modes and hand each to a separate context. This is rare.

## Why not a timestamp?

A common alternative to a version integer is a `TIMESTAMP` / `ROWVERSION` column — SQL Server's built-in, auto-incremented row version. Trysil doesn't use that for two reasons:

1. **Portability.** SQLite, PostgreSQL, and Firebird don't have an exact equivalent. The closest is a triggered timestamp, which introduces a database-specific dependency that `[TVersionColumn]` was designed to avoid.
2. **Determinism.** Two updates happening in the same millisecond would share a timestamp and defeat the purpose. A monotonically incrementing integer is unambiguous.

If you're on pure SQL Server and want to use `ROWVERSION` specifically, you can — map it as a `TTVersion` column and let SQL Server increment it server-side, and tell Trysil to use `KeyAndVersionColumn`. But the portable default is an application-managed integer.

## Soft delete and versioning together

The two features compose. An entity with both `[TDeletedAt]` and `[TVersionColumn]` gets:

- `Delete` becomes `UPDATE` (soft delete).
- That `UPDATE` carries the version predicate (optimistic locking) and increments the version.

So a stale "delete" fails the same way as a stale "update". Two users can't both soft-delete the same row from two stale copies — whichever commits second sees the conflict.

## Closing

One attribute, one column, one type. The database becomes the arbiter of who saw the row most recently, without any long-held locks, without a central coordinator, and without any opinions in the ORM beyond "the row must be as you last saw it". It costs one small integer per row and one line in the `WHERE` clause. There is no good reason not to have it on every writable table.

Next: `TTLazy<T>` and `TTLazyList<T>` — what Trysil does instead of auto-joining related entities, and when lazy loading is actually what you want.
