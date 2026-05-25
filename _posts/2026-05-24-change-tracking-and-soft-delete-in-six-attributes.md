---
title: "Change tracking and soft delete in six attributes"
date: 2026-05-24 07:47:00 +0200
categories: [Deep Dives]
tags: [change-tracking, soft-delete, attributes]
---

Every application eventually needs the same four things on its important tables: *when was this row created, by whom, when was it last updated, by whom*. Some applications also want a fifth and sixth: *don't actually delete rows — flag them, and record when and by whom they were flagged*.

This is boilerplate. It's the same six columns on a hundred tables, the same six assignments in a hundred service methods, the same six mistakes waiting to happen when someone forgets one.

Trysil offers six attributes that eliminate all of it. Decorate the right columns, set one callback, and the framework does the rest.

## The six attributes

They live in `Trysil.Attributes.pas` and come in three pairs:

| Attribute | Fires on | Column type |
|---|---|---|
| `[TCreatedAt]` | `Insert` | `TTNullable<TDateTime>` |
| `[TCreatedBy]` | `Insert` | `String` |
| `[TUpdatedAt]` | `Update` | `TTNullable<TDateTime>` |
| `[TUpdatedBy]` | `Update` | `String` |
| `[TDeletedAt]` | `Delete` | `TTNullable<TDateTime>` |
| `[TDeletedBy]` | `Delete` | `String` |

The `*At` columns are `TTNullable<TDateTime>` because a just-created row has no `UpdatedAt` yet, and a row that's never been deleted has no `DeletedAt`. Null is the truthful answer.

The `*By` columns are plain `String`. Nullable string would also work, but plain string lets you default to `''` without ceremony, and an empty-string author is as good as null for any query you'd write.

## A tracked entity

```pascal
[TTable('Orders')]
TOrder = class
strict private
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TColumn('CustomerID')]
  FCustomerID: TTPrimaryKey;

  [TColumn('Amount')]
  FAmount: Double;

  [TCreatedAt]
  [TColumn('CreatedAt')]
  FCreatedAt: TTNullable<TDateTime>;

  [TCreatedBy]
  [TColumn('CreatedBy')]
  FCreatedBy: String;

  [TUpdatedAt]
  [TColumn('UpdatedAt')]
  FUpdatedAt: TTNullable<TDateTime>;

  [TUpdatedBy]
  [TColumn('UpdatedBy')]
  FUpdatedBy: String;

  [TDeletedAt]
  [TColumn('DeletedAt')]
  FDeletedAt: TTNullable<TDateTime>;

  [TDeletedBy]
  [TColumn('DeletedBy')]
  FDeletedBy: String;
public
  property ID: TTPrimaryKey read FID;
  property CustomerID: TTPrimaryKey read FCustomerID write FCustomerID;
  property Amount: Double read FAmount write FAmount;
  property CreatedAt: TTNullable<TDateTime> read FCreatedAt;
  property CreatedBy: String read FCreatedBy;
  property UpdatedAt: TTNullable<TDateTime> read FUpdatedAt;
  property UpdatedBy: String read FUpdatedBy;
  property DeletedAt: TTNullable<TDateTime> read FDeletedAt;
  property DeletedBy: String read FDeletedBy;
end;
```

Note the read-only properties on the tracking fields — callers shouldn't write these by hand. The resolver does.

## Who is the current user?

The `*At` attributes know the timestamp — `Now`. The `*By` attributes can't know the user on their own, so Trysil asks you:

```pascal
LContext.OnGetCurrentUser :=
  function: String
  begin
    result := GlobalSession.UserName;
  end;
```

Set it once on the context. Every `Insert`, `Update`, and `Delete` call through that context will invoke the callback when it needs to fill a `[T*By]` column. If you don't set it, Trysil writes an empty string — valid, but unhelpful.

In a web app, this is where the HTTP pipeline plugs in: the authentication middleware decodes the JWT, drops the username into a request-scoped field, and `OnGetCurrentUser` reads from there. We'll come back to that when we get to the HTTP module.

## Soft delete

The pair that changes behavior, not just values, is `[TDeletedAt]`. When Trysil sees it on an entity, `Delete<T>` stops issuing SQL `DELETE`. It issues an `UPDATE` instead:

```sql
UPDATE Orders
   SET DeletedAt = CURRENT_TIMESTAMP,
       DeletedBy = 'alice',
       VersionID = VersionID + 1
 WHERE ID = 42
   AND VersionID = 7
```

The version column still increments (optimistic locking isn't skipped), but the row stays. From the database's perspective, nothing was deleted.

From the application's perspective, the row appears gone — because every `SELECT` Trysil generates for entities with `[TDeletedAt]` adds an implicit clause:

```sql
SELECT ... FROM Orders WHERE DeletedAt IS NULL
```

This is the part that makes soft delete actually work. If you had to remember to add `AND DeletedAt IS NULL` to every query, you'd forget, and soft-deleted rows would start leaking into production reports. Trysil adds it automatically. You get the database hygiene of *keep everything* with the ergonomics of *deleted means gone*.

One subtlety: when the entity has JOINs, the `DeletedAt IS NULL` is qualified with the FROM table name — `Orders.DeletedAt IS NULL`, never ambiguous across joins.

## Turning soft delete off (per query)

Sometimes you want the soft-deleted rows back. Audit logs, restore flows, admin UI for trash-bin views. Both `TTFilter` and `TTFilterBuilder<T>` have an opt-in:

```pascal
LFilter := TTFilter.Create('');
LFilter.IncludeDeleted := True;
LContext.Select<TOrder>(LList, LFilter);
```

Or with the builder:

```pascal
LFilter := LBuilder
  .Where('CustomerID').Equal(42)
  .IncludeDeleted
  .Build;
```

Now the generated SQL omits the `DeletedAt IS NULL` clause and you see everything.

## Relation checks bypass soft delete

Before hard-deleting a row, Trysil asks every table that has a foreign key to it: *is anyone pointing at this?* If yes, the delete raises `ETException` — no cascade, no orphan rows.

Soft delete doesn't ask. The row isn't actually leaving, so referential integrity is still intact at the database level. `CheckRelations` is skipped entirely for soft-deleted entities. This is what you want: you can "delete" a customer with orders pointing at it, and the orders keep resolving their `CustomerID` to a row that's flagged but still there.

## Why these attributes, not events

You could do all this with `[BeforeInsert]` / `[BeforeUpdate]` / `[BeforeDelete]` event hooks. Write six methods, assign six fields, done.

The reason to use attributes instead is that the intent is *declarative*. `[TCreatedAt]` doesn't describe what to run — it describes what the column *is*. The resolver reads the attribute, knows what to write, and a second reader of the code sees the role of each column without chasing method implementations.

Events are the escape hatch for anything that doesn't fit one of the six shapes — computed totals, audit logs, cross-table side effects. We'll look at those in the post on events, later in the series.

## Closing

Six attributes, one callback, one filter flag. Every tracked table behaves the same; every service method writes zero boilerplate. The database keeps the history; the application sees only the live rows unless it asks otherwise.

Next: the abstraction that lets the *same* entity class run on SQLite, PostgreSQL, Firebird, and SQL Server without a single line of code change.
