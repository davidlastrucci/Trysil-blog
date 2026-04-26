---
title: "JOIN queries in Trysil"
date: 2026-04-26 08:30:00 +0200
categories: [Deep Dives]
tags: [joins, entity-mapping, attributes, internals]
---

You have an `Orders` table. Each row references a `Customer` by ID. You want to list orders with the customer's company name on the row — one query, one result set, no round trips.

Traditional ORM answers:

- **Relations with lazy loading.** Map `TOrder.CustomerID` as a `TTLazy<TCustomer>` and trust that each access fetches the customer. 100 orders, 100 customer lookups — the N+1 problem in its purest form.
- **Raw SQL.** Write `SELECT o.ID, c.CompanyName FROM Orders o JOIN Customers c ON ...` yourself. Trysil's `RawSelect<T>` handles this fine, but you lose the mapping layer — no attribute-driven discovery, no consistency with the rest of your entities.

Since 2026, Trysil has a third option: **attribute-based JOIN queries**. You declare the join on the class, Trysil generates the SQL, and the resulting entity looks and behaves like any other — except it's read-only.

This post walks through the three forms and the design decisions behind them.

## The simple form

Here's the orders-with-customer-name case:

```pascal
[TTable('Orders')]
[TJoin(TJoinKind.Inner, 'Customers', 'CustomerID', 'ID')]
TOrderReport = class
strict private
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TColumn('Amount')]
  FAmount: Double;

  [TColumn('Customers', 'CompanyName')]
  FCustomerName: String;
public
  property ID: TTPrimaryKey read FID;
  property Amount: Double read FAmount;
  property CustomerName: String read FCustomerName;
end;
```

Two new attributes:

- **`[TJoin(Kind, 'Table', 'SourceCol', 'TargetCol')]`** — one attribute per joined table. Here: `INNER JOIN Customers ON Orders.CustomerID = Customers.ID`. `TJoinKind` is a scoped enum with values `Inner`, `Left`, `Right`.
- **`[TColumn('Customers', 'CompanyName')]`** — the two-parameter form. The first arg is the join alias (same as the joined table name when there's no explicit alias), the second is the column in that table.

Query it with the same API as any other entity:

```pascal
var
  LOrders: TTList<TOrderReport>;
  LOrder: TOrderReport;
begin
  LOrders := LContext.CreateEntityList<TOrderReport>();
  try
    LContext.SelectAll<TOrderReport>(LOrders);
    for LOrder in LOrders do
      Writeln(Format('%d: %s — %.2f',
        [LOrder.ID, LOrder.CustomerName, LOrder.Amount]));
  finally
    LOrders.Free;
  end;
end;
```

The generated SQL:

```sql
SELECT
  Orders.ID        AS Orders_ID,
  Orders.Amount    AS Orders_Amount,
  Customers.CompanyName AS Customers_CompanyName
FROM Orders
INNER JOIN Customers ON Orders.CustomerID = Customers.ID
```

One row per order. No N+1.

## The aliased form (self-joins)

Some queries need to join the same table twice. Accounting is the classic example: a journal entry (`Movimenti`) has a debit account and a credit account, both pointing at the same chart-of-accounts table (`PianoDeiConti`). You want the account name for each leg, on one row.

```pascal
[TTable('Movimenti')]
[TJoin(TJoinKind.Inner, 'PianoDeiConti', 'ContoDare',  'ContoDareID',  'ID')]
[TJoin(TJoinKind.Inner, 'PianoDeiConti', 'ContoAvere', 'ContoAvereID', 'ID')]
TMovimentoContabile = class
strict private
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TColumn('ContoDare', 'Descrizione')]
  FDescrizioneDare: String;

  [TColumn('ContoAvere', 'Descrizione')]
  FDescrizioneAvere: String;
end;
```

The five-parameter `TJoin` overload introduces an explicit alias as the third arg: `'ContoDare'` and `'ContoAvere'`. The generated SQL uses those aliases:

```sql
... INNER JOIN PianoDeiConti AS ContoDare  ON Movimenti.ContoDareID  = ContoDare.ID
    INNER JOIN PianoDeiConti AS ContoAvere ON Movimenti.ContoAvereID = ContoAvere.ID
```

The `[TColumn('ContoDare', 'Descrizione')]` on the field picks which of the two aliases the column comes from. No SQL written by hand.

## The chained form

Sometimes a join depends on another join: `A → B → C`, where `C` is reached through `B`, not directly from `A`.

```pascal
[TTable('Orders')]
[TJoin(TJoinKind.Inner, 'Customers', 'CustomerID', 'ID')]
[TJoin(TJoinKind.Left,  'Countries', 'Countries', 'Customers', 'CountryID', 'ID')]
TOrderWithCountry = class
strict private
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TColumn('Customers', 'CompanyName')]
  FCustomerName: String;

  [TColumn('Countries', 'Name')]
  FCountryName: String;
end;
```

The six-parameter `TJoin` overload names the **source** table/alias as the fourth argument — here `'Customers'`, meaning *"join Countries to Customers, not to Orders"*. The generated SQL:

```sql
... FROM Orders
    INNER JOIN Customers ON Orders.CustomerID = Customers.ID
    LEFT  JOIN Countries ON Customers.CountryID = Countries.ID
```

Chains can go arbitrarily deep, as long as every non-first join names a previously-declared alias as its source.

## How the column names resolve

For single-table entities, `[TColumn('Firstname')]` maps to a SQL column literally called `Firstname`. For join entities, Trysil does something subtly different: internally the column name becomes `Alias_ColumnName` — `Customers_CompanyName` in the first example. This aliased form is what the SELECT emits, what the mapper reads from the result set, and what `TTColumnMap.LookupName` returns.

All of this is transparent. From user code you never see `Customers_CompanyName`. The attribute says `('Customers', 'CompanyName')`, the property is `CustomerName`, and Trysil handles the SQL shape in between.

## Why read-only

Join entities raise `ETException` if you try to `Insert`, `Update`, or `Delete` them. The reason is that a join result is a projection over multiple tables — *"which row do I write back to?"* has no single answer. If you need to modify the underlying data, load the single-table entities for each row you care about and mutate those.

Two other consequences of the read-only choice:

- **Identity map is skipped for join entities.** The same primary key can appear in multiple result rows (one per joined child combination), so caching by PK would collapse rows incorrectly. Join entities always come back fresh.
- **Soft-delete stays on the FROM table.** When Trysil adds `DeletedAt IS NULL` (because the FROM entity has a `[TDeletedAt]` column), the column is qualified with the FROM table name — `Orders.DeletedAt IS NULL`, never ambiguous across joins.

## Current limitations

`TTFilterBuilder<T>` doesn't resolve join aliases yet. If you want a dynamic `WHERE` clause against a join entity, use `TTFilter.Create(whereClause)` directly and write column references with explicit table qualification:

```pascal
LFilter := TTFilter.Create('Customers.CompanyName LIKE ?', ['Smith%']);
LContext.Select<TOrderReport>(LOrders, LFilter);
```

`[TWhereClause]` on a join entity has the same constraint: manual qualification required. Lifting it means teaching the builder to consult `TTColumnMap.LookupName` the same way the SELECT generator does — on the roadmap.

## When attributes aren't enough

Some queries aren't joins in the strict sense. Subqueries, `UNION`, `GROUP BY`, aggregations — `[TJoin]` can't express them. For those, `RawSelect<T>` remains the escape hatch:

```pascal
LContext.RawSelect<TOrderSummary>(
  'SELECT c.CompanyName AS CustomerName, SUM(o.Amount) AS Total ' +
  'FROM Orders o JOIN Customers c ON o.CustomerID = c.ID ' +
  'GROUP BY c.CompanyName',
  LResult);
```

A DTO with `[TColumn]` attributes matching the result column names, raw SQL, read-only results. No attribute-driven discovery, but full SQL power.

## Closing

Join queries give Trysil entities a way to carry data from multiple tables without giving up the attribute-driven model. They're read-only by design, the column aliasing is transparent, and anything they can't express falls back cleanly to `RawSelect<T>`.

If you've been reaching for `TTLazy<T>` for what should really be a single SELECT with a join, `[TJoin]` is your next stop.
