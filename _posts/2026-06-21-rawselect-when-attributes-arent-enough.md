---
title: "RawSelect: when attributes aren't enough"
date: 2026-06-21 07:58:00 +0200
categories: [Tutorials]
tags: [raw-sql, dto, aggregations, tutorial]
---

Attribute-driven SQL is where most ORMs earn their keep. `[TTable]`, `[TColumn]`, `[TJoin]`, a filter — and the framework hands you back typed entities. It covers 80% of real queries.

The remaining 20% is where reality gets harder. Aggregations. Window functions. Subqueries. `UNION`. `GROUP BY` with `HAVING`. Queries that return shapes that don't correspond to any table.

For that 20%, Trysil gives you `RawSelect<T>`. You write the SQL; Trysil maps the result rows to a DTO class by column name.

## A summary query

A concrete case: summing order amounts by customer, for a sales dashboard. There's no `OrderSummary` table — the row you want doesn't exist until you compute it. You can't express this with `[TJoin]` because the output isn't *a row per order joined with a customer*, it's *one row per customer, with an aggregate across their orders*.

Start with a DTO that matches the shape you want back:

```pascal
TOrderSummary = class
strict private
  [TColumn('CustomerName')]
  FCustomerName: String;

  [TColumn('OrderCount')]
  FOrderCount: Integer;

  [TColumn('Total')]
  FTotal: Double;
public
  property CustomerName: String read FCustomerName;
  property OrderCount: Integer read FOrderCount;
  property Total: Double read FTotal;
end;
```

Three things to notice:

- **No `[TTable]`.** This class is never persisted, so it has no table to map to. Trysil doesn't require one for `RawSelect`.
- **No `[TPrimaryKey]`.** Same reason. There is no primary key on an aggregate row.
- **No `[TSequence]`.** Also unnecessary.

Only `[TColumn('name')]` is needed, and the name must match the SQL result column exactly (case-insensitive on every supported database).

Then execute:

```pascal
var
  LList: TTList<TOrderSummary>;
begin
  LList := LContext.CreateEntityList<TOrderSummary>();
  try
    LContext.RawSelect<TOrderSummary>(
      'SELECT c.CompanyName   AS CustomerName,   ' +
      '       COUNT(o.ID)     AS OrderCount,     ' +
      '       SUM(o.Amount)   AS Total           ' +
      'FROM Orders o                             ' +
      'JOIN Customers c ON o.CustomerID = c.ID   ' +
      'GROUP BY c.CompanyName                    ' +
      'ORDER BY Total DESC',
      LList);
    // ... use LList
  finally
    LList.Free;
  end;
end;
```

No parameters, pure SQL, DTOs hydrated from the result set.

## Parameters

For anything involving user input, never concatenate — `RawSelect` has an overload that takes a filter:

```pascal
var
  LFilter: TTFilter;
begin
  LFilter := TTFilter.Create('');
  LFilter.AddParameter('since', ftDateTime, TTValue.From<TDateTime>(LLastMonth));

  LContext.RawSelect<TOrderSummary>(
    'SELECT c.CompanyName AS CustomerName, ' +
    '       COUNT(o.ID)   AS OrderCount,   ' +
    '       SUM(o.Amount) AS Total         ' +
    'FROM Orders o                         ' +
    'JOIN Customers c ON o.CustomerID = c.ID ' +
    'WHERE o.CreatedAt >= :since           ' +
    'GROUP BY c.CompanyName',
    LFilter,
    LList);
end;
```

The `TTFilter` record here carries parameters only — the WHERE clause is already baked into your SQL, since `RawSelect` doesn't try to rewrite the query. Trysil binds the parameters through FireDAC the same way it does for attribute-driven queries; SQL injection isn't a risk if you stick to named parameters.

## What RawSelect does not do

The trade-off for raw SQL is that several things Trysil normally gives you are gone:

- **No identity map.** Two `RawSelect` calls returning the same logical row produce two distinct DTO instances. That's correct for DTOs — they carry values, not identity — but worth remembering.
- **No lazy loading.** If your DTO has a property you'd normally fill with `TTLazy<T>`, it stays null. `RawSelect` only populates plain fields mapped with `[TColumn]`.
- **No writing back.** You can't `Insert`, `Update`, or `Delete` a DTO. They're read-only by construction.
- **No soft-delete filter.** If you want `DeletedAt IS NULL`, you write it yourself. Trysil only injects that clause in attribute-driven queries where it knows the column exists.
- **No optimistic locking.** Irrelevant for read-only DTOs, but noted here for completeness.

In exchange, you can write anything SQL can express.

## A subquery shape

Subqueries that would be awkward to build with `[TJoin]` become trivial:

```pascal
TOverdueCustomer = class
strict private
  [TColumn('CustomerName')]
  FCustomerName: String;

  [TColumn('OverdueAmount')]
  FOverdueAmount: Double;
public
  property CustomerName: String read FCustomerName;
  property OverdueAmount: Double read FOverdueAmount;
end;
```

```pascal
LContext.RawSelect<TOverdueCustomer>(
  'SELECT c.CompanyName AS CustomerName, ' +
  '       (SELECT SUM(i.Amount)          ' +
  '          FROM Invoices i             ' +
  '         WHERE i.CustomerID = c.ID    ' +
  '           AND i.PaidAt IS NULL       ' +
  '           AND i.DueDate < :today) AS OverdueAmount ' +
  'FROM Customers c                      ' +
  'WHERE c.Active = 1',
  LFilter,
  LList);
```

The correlated subquery is hand-written — which is *the right answer*, because the subquery's performance characteristics are specific to the query planner of whichever database is behind the connection. An ORM that tried to generate this from attributes would be inventing opinions it doesn't have.

## A UNION shape

```pascal
TActivityRow = class
strict private
  [TColumn('EventKind')]
  FEventKind: String;

  [TColumn('Happened')]
  FHappened: TDateTime;

  [TColumn('Description')]
  FDescription: String;
public
  property EventKind: String read FEventKind;
  property Happened: TDateTime read FHappened;
  property Description: String read FDescription;
end;
```

```sql
SELECT 'Order'   AS EventKind, o.CreatedAt AS Happened, o.Description
  FROM Orders o
 WHERE o.CustomerID = :customerId
UNION ALL
SELECT 'Invoice' AS EventKind, i.IssuedAt  AS Happened, i.Description
  FROM Invoices i
 WHERE i.CustomerID = :customerId
 ORDER BY Happened DESC
```

Pass it to `RawSelect<TActivityRow>` with a parameter of `customerId`, and you've got a customer activity timeline in one round trip.

## DTOs vs. entities: when to reach for which

The rule I use:

- **Entity with `[TJoin]`** when the output corresponds to a row of a base table plus columns pulled from joined tables. Read-only, yes, but still *of* the base table.
- **DTO with `RawSelect`** when the output is a *computed shape* that doesn't correspond to any one table — aggregations, unions, pivoted data, anything that involves `GROUP BY` or window functions.

If you catch yourself writing raw SQL for something that could be expressed with `[TJoin]`, reach for the join instead — you get typed entities and automatic soft-delete filtering for free. If you catch yourself fighting `[TJoin]` to express something it wasn't meant for, stop and write the SQL.

## Closing

`RawSelect<T>` is the escape hatch — it exists precisely so you don't have to fight the ORM when the ORM is the wrong tool. DTOs keep the hand-written SQL lane well-scoped: values in, objects out, no write-back surprise. Attributes handle the common case; SQL handles everything else; Trysil doesn't pretend either one is universal.

Next: validation attributes, and the method-level hooks for anything validation attributes can't express.
