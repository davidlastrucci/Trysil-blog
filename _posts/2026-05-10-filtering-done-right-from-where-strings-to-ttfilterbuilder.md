---
title: "Filtering done right: from WHERE strings to TTFilterBuilder"
date: 2026-05-10 05:12:00 +0200
categories: [Tutorials]
tags: [filtering, filter-builder, fluent-api, tutorial]
---

In the first post we fetched every row with `SelectAll<TPerson>`. That's fine when the table has ten rows. At ten thousand, it isn't.

This post walks through the three ways Trysil lets you filter, in order of increasing power:

1. A `TTFilter` with a raw WHERE string — dead simple, good for one-off queries.
2. Parameters on that same filter — same simplicity, no SQL injection.
3. `TTFilterBuilder<T>` — a fluent API that knows your entity and refuses typos at runtime.

We'll use the `TPerson` entity from the first post. No new setup, same SQLite file.

## The raw form

`TTFilter` is a record. The one-argument constructor takes a SQL `WHERE` clause and that's it:

```pascal
var
  LPeople: TTList<TPerson>;
  LFilter: TTFilter;
begin
  LPeople := LContext.CreateEntityList<TPerson>();
  try
    LFilter := TTFilter.Create('Age >= 18');
    LContext.Select<TPerson>(LPeople, LFilter);
    // ... use LPeople
  finally
    LPeople.Free;
  end;
end;
```

`Select<T>` takes the list and the filter, runs `SELECT ... FROM Person WHERE Age >= 18`, maps the rows into `TPerson` instances, and adds them to the list. The filter record is a value type — you can create it inline, pass it around, and forget about freeing it.

## Parameters

The raw form is fine for constants, but the moment you concatenate user input into that string you've just built a SQL injection. Use parameters:

```pascal
LFilter := TTFilter.Create('Firstname LIKE :name');
LFilter.AddParameter('name', ftString, TTValue.From<String>('A%'));
LContext.Select<TPerson>(LPeople, LFilter);
```

`AddParameter(name, dataType, value)` registers a bind parameter. The name in the SQL (`:name`) matches the name passed to `AddParameter`. The `ftString` comes from FireDAC's `Data.DB.TFieldType`, and `TTValue.From<T>` boxes the typed value into the variant-like `TTValue` that Trysil uses internally.

There's a four-argument overload that also takes the field size if you need it for `ftFixedChar` or similar.

Paging comes from a longer constructor:

```pascal
LFilter := TTFilter.Create('Age >= :min', 0, 20, 'Lastname');
LFilter.AddParameter('min', ftInteger, TTValue.From<Integer>(18));
```

Arguments: WHERE clause, start offset, row limit, ORDER BY clause. The three-argument overload (`WHERE, MaxRecord, OrderBy`) is a shortcut that sets start to 0.

And if you want soft-deleted rows in the result, flip the flag after construction:

```pascal
LFilter.IncludeDeleted := True;
```

## The builder

The raw filter works, but it has problems that show up slowly:

- **Column names are strings.** If you rename `Firstname` to `FirstName` in the entity, the filter keeps compiling and blows up at query time with a SQL error that mentions a column Trysil doesn't know about.
- **Parameter types are spelled out.** `ftString`, `ftInteger`, `ftDateTime`. The entity already knows the types — you're repeating yourself.
- **Operators are SQL.** `LIKE`, `=`, `<>`, `IS NULL`. Fine if you already know SQL, noise if you're reading the Delphi code months later.

`TTFilterBuilder<T>` fixes all three. You obtain one through the context:

```pascal
var
  LBuilder: TTFilterBuilder<TPerson>;
  LFilter: TTFilter;
begin
  LBuilder := LContext.CreateFilterBuilder<TPerson>();
  try
    LFilter := LBuilder
      .Where('Firstname').Like('A%')
      .AndWhere('Age').GreaterOrEqual(18)
      .OrderByAsc('Lastname')
      .Limit(20)
      .Build;
    LContext.Select<TPerson>(LPeople, LFilter);
  finally
    LBuilder.Free;
  end;
end;
```

The builder looks up the column metadata at construction time. If you pass `'Firstame'` (typo), you get `ETException` immediately — not a SQL error three rebuilds later.

Conditions chain through a small per-column object (`TTFilterCondition<T>`) that exposes the operators:

| Method | SQL |
|---|---|
| `Equal(v)` | `= ?` |
| `NotEqual(v)` | `<> ?` |
| `Greater(v)` | `> ?` |
| `GreaterOrEqual(v)` | `>= ?` |
| `Less(v)` | `< ?` |
| `LessOrEqual(v)` | `<= ?` |
| `Like(s)` | `LIKE ?` |
| `NotLike(s)` | `NOT LIKE ?` |
| `IsNull` | `IS NULL` |
| `IsNotNull` | `IS NOT NULL` |

`Where` / `AndWhere` / `OrWhere` govern how a new condition joins the previous one. The builder internally numbers parameters `p0`, `p1`, ... so you don't.

Ordering and paging are one-liners, and the builder is self-referential so every method returns the builder:

```pascal
LFilter := LBuilder
  .Where('Active').Equal(True)
  .OrWhere('CreatedAt').Greater(LLastWeek)
  .OrderByDesc('CreatedAt')
  .Offset(40)
  .Limit(20)
  .IncludeDeleted
  .Build;
```

`Build` produces a `TTFilter` record that you feed to `Select`. The builder itself is freed when you're done — it owns its internal condition objects.

## What the builder doesn't do

Two limitations to keep in mind.

**Grouping.** There's no `BeginGroup` / `EndGroup`. If you need `(A AND B) OR (C AND D)`, the builder can't express it — left-to-right chaining produces `A AND B OR C AND D`, which SQL will evaluate `A AND (B OR C) AND D`. For anything non-trivial, fall back to `TTFilter.Create(whereClause)` with hand-written SQL and parameters.

**Joins.** The builder only knows columns on the FROM table. If you have a `[TJoin]` entity (see the previous post), `AndWhere('Customers_CompanyName')` works against Trysil's internal aliased name, but there's no validation that the column exists on the joined table. This is on the roadmap.

## One filter, many calls

A `TTFilter` record is cheap. Pass it to `Select` for the rows, pass the same record to `SelectCount` for the total:

```pascal
LFilter := LBuilder
  .Where('Age').GreaterOrEqual(18)
  .OrderByAsc('Lastname')
  .Limit(20)
  .Build;

LTotal := LContext.SelectCount<TPerson>(LFilter);
LContext.Select<TPerson>(LPeople, LFilter);
```

This is how the HTTP controllers in Trysil.Http build paginated endpoints: one filter, a count for the page counter, a select for the current page. You'll see that pattern again when we get to the HTTP module later in the series.

## Closing

`TTFilter.Create(whereClause)` is the short answer for quick queries. `TTFilterBuilder<T>` is the right answer for anything that outlives the method it's written in — it validates columns at query-build time, picks the right field type from metadata, and gives you a name for every operator. Pick the tool that fits the scope; Trysil has both on purpose.

Next post: change tracking and soft delete — six attributes that make your tables remember who touched what, and when.
