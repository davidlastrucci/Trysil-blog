---
title: "One codebase, many databases: the Trysil driver abstraction"
date: 2026-06-07 08:33:00 +0200
categories: [Design Notes]
tags: [drivers, polymorphism, multi-database]
---

Delphi has FireDAC. FireDAC already talks to SQLite, PostgreSQL, Firebird, SQL Server, MySQL, Oracle, and a handful of others. If connectivity is a solved problem, why does Trysil need a driver layer at all?

The answer is that connectivity is not the problem. *SQL generation* is. Every database disagrees with every other one on the details that matter to an ORM:

- How do you fetch a row by primary key, paged?
- How do you ask an `IDENTITY` / `SERIAL` / `AUTOINCREMENT` / `GENERATED` column for the ID it just assigned?
- How do you get the next value from a named sequence?
- What does `NOW()` look like — `CURRENT_TIMESTAMP`, `GetDate()`, `NOW()`?

FireDAC papers over connection protocols. It does not paper over these. Trysil does.

This post walks through how that abstraction is shaped, what changes when you switch drivers, and — crucially — what *doesn't*.

## What the consumer sees

The first post in this blog ended with a line like this:

```pascal
var
  LConnection: TTConnection;
begin
  TTSQLiteConnection.RegisterConnection('Main', 'persons.db');
  LConnection := TTSQLiteConnection.Create('Main');
```

Swap `TTSQLiteConnection` for `TTPostgreSQLConnection` or `TTFirebirdSQLConnection` or `TTSqlServerConnection`, adjust the `RegisterConnection` arguments to match the target, and your code is done. Everything downstream — `TTContext`, every `Select`, every `Insert`, every entity class — stays unchanged.

The variable type `TTConnection` matters. Declare it as the concrete driver class and you're coupled; declare it as the polymorphic base and the compiler stops caring which driver you chose. In production code, always the base.

## What TTConnection actually abstracts

`TTConnection` (`Trysil.Data.pas`) is not a thin wrapper over a FireDAC connection. It's a contract that every driver must fulfil:

```pascal
TTConnection = class abstract
  // Transactions
  procedure StartTransaction; virtual; abstract;
  procedure CommitTransaction; virtual; abstract;
  procedure RollbackTransaction; virtual; abstract;
  function InTransaction: Boolean; virtual; abstract;

  // Queries
  function CreateReader(
    const ATableMap: TTTableMap;
    const AFilter: TTFilter): TTAbstractDataReader; virtual; abstract;
  function CreateInsertCommand(...): TTDataAbstractCommand; virtual; abstract;
  function CreateUpdateCommand(...): TTDataAbstractCommand; virtual; abstract;
  function CreateDeleteCommand(...): TTDataAbstractCommand; virtual; abstract;
  function CreateSoftDeleteCommand(...): TTDataAbstractCommand; virtual; abstract;

  // Identity & sequences
  function GetSequenceID(const ASequenceName: String): TTPrimaryKey; virtual; abstract;

  // Integrity
  function SelectCount(...): Integer; virtual; abstract;
  procedure CheckRelations(...); virtual; abstract;
end;
```

Each of the four drivers extends this through `TTGenericConnection` (which adds a per-instance UUID for log correlation) and plugs in a database-specific **syntax strategy**.

## The syntax strategy

The ugliest parts of any ORM are pagination, identity retrieval, and sequences. In Trysil they live in `Trysil/Data/SqlSyntax/`, one file per database:

```
Trysil.Data.SqlSyntax.pas              -- base classes
Trysil.Data.SqlSyntax.SQLite.pas
Trysil.Data.SqlSyntax.PostgreSQL.pas
Trysil.Data.SqlSyntax.FirebirdSQL.pas
Trysil.Data.SqlSyntax.SqlServer.pas
```

Each driver instantiates its own `TTSyntaxClasses` — a bundle of small classes that know, for that database:

- How to shape a `SELECT` with `LIMIT`/`OFFSET` (PostgreSQL, SQLite), `FIRST`/`SKIP` (Firebird), or `OFFSET ... FETCH NEXT` with an `ORDER BY` requirement (SQL Server).
- How to ask for the just-inserted identity: `RETURNING id` (PostgreSQL, Firebird 3+), `SCOPE_IDENTITY()` (SQL Server), `last_insert_rowid()` (SQLite).
- How to query a sequence — which on SQLite means "simulate with a tiny helper table" because SQLite doesn't have real sequences.
- How to emit `UPDATE`/`DELETE` commands for soft delete, with the right `CURRENT_TIMESTAMP` literal.

The `TTContext` never sees any of this. It talks to `TTConnection`. `TTConnection` talks to its syntax strategy. The strategy emits SQL that the underlying FireDAC connection executes.

## What FireDAC does, what Trysil does

| Concern | FireDAC | Trysil |
|---|---|---|
| TCP/pipes/IPC to the database | yes | no |
| Parameter binding & data marshalling | yes | no |
| Connection pooling | yes (wrapped by `TTFireDACConnectionPool`) | no |
| SELECT with paging | generic | per-driver SQL generation |
| INSERT + identity retrieval | generic | per-driver SQL generation |
| Sequence access | generic | per-driver (SQLite falls back to helper table) |
| Optimistic locking `WHERE VersionID = ?` | no | always |
| Soft delete rewriting DELETE as UPDATE | no | yes |

The split is deliberate: FireDAC is good at wire protocols and weakly opinionated about SQL; Trysil is strongly opinionated about SQL and useless at wire protocols. They compose.

## A switching test

Because the abstraction is real, not paper-thin, Trysil's own test suite runs against four databases from the same test files. A typical test fixture registers one connection type, and the same 404 tests run — no conditional compilation, no test forks.

```pascal
// conftest-style setup — one of four implementations
function CreateConnection: TTConnection;
begin
  result := TTSQLiteConnection.Create('Main');
  // or TTPostgreSQLConnection.Create('Main')
  // or TTFirebirdSQLConnection.Create('Main')
  // or TTSqlServerConnection.Create('Main')
end;
```

The test that calls `Select`, `Insert`, `Update`, `Delete` sees exactly the same behavior across all four — different SQL under the hood, identical results on top. If the abstraction leaked, the test suite would be the first thing to crack.

## What stays the same, in one list

- Entity class declarations. `[TTable]`, `[TColumn]`, `[TPrimaryKey]`, `[TVersionColumn]`, `[TSequence]` — all driver-agnostic.
- Change tracking attributes. Soft delete works the same everywhere.
- `TTFilter` and `TTFilterBuilder<T>` output the same WHERE shape.
- `TTSession<T>` clones the same way.
- `[TJoin]` produces the same SQL `JOIN` on every driver (the variance is in paging around the joined statement, which the syntax class handles).

## What changes, in one list

- The `RegisterConnection` arguments — each driver takes its own parameters (filename vs. host/port/database).
- The driver class name — `TTSQLiteConnection` vs `TTPostgreSQLConnection` vs …
- The FireDAC link units you `uses` (`FireDAC.Phys.SQLite` vs `FireDAC.Phys.PG` and so on).
- The edition requirement — `Trysil.SqlServer` needs Delphi Enterprise; the other three drivers compile on Community.

That's it. No entities to rewrite, no queries to rewrite, no transactions to think about.

## Why this matters

Most real projects pick a database once and never switch. The value of the abstraction isn't that you *will* switch — it's that you *can*. Development in SQLite, integration tests in SQLite in-memory, staging in the real database, production wherever the customer has a DBA. One codebase across the four environments, no ceremony.

It also makes consulting work tractable. When the customer's preferred database isn't the one you built the framework on, you don't refuse the job — you add a driver module. The surface area is small because everything above `TTConnection` doesn't care.

## Closing

The Trysil driver layer is deliberately thin: it only owns the parts of SQL that databases disagree on, and delegates everything else to FireDAC. The payoff is that `TTConnection` is a real boundary — if you declare your variable against the base type, the rest of your application is database-agnostic by construction.

Next post: `RawSelect` — what to do when attribute-driven queries can't express what you need.

> **Update — 2026-06-08.** Three more drivers have shipped: **InterBase**, **MariaDB**, and **Oracle**. Trysil now covers seven databases — Firebird, InterBase, MariaDB, Oracle, PostgreSQL, SQL Server, and SQLite — through the exact same `TTConnection` boundary described above. Nothing in this post changes: each new driver is just another `TTGenericConnection` subclass with its own syntax strategy, and the code above `TTConnection` doesn't know the difference.
{: .prompt-info }
