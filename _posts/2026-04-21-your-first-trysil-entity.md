---
title: "Your first Trysil entity"
date: 2026-04-21 07:42:00 +0200
categories: [Tutorials]
tags: [entity-mapping, attributes, sqlite, tutorial]
---

The ORM story is always the same: you have a table, you have a class, and you want them to know about each other without writing a thousand lines of glue code. Trysil's answer is **attributes** — small decorations on your class fields that tell the framework how to talk to the database.

This post walks through the smallest possible Trysil program: define one entity, read rows, insert, update, delete. We'll use SQLite to keep the setup friction minimal — no server to install, just a file.

## The table

Assume you have an SQLite database with a single table:

```sql
CREATE TABLE Persons (
  ID        INTEGER PRIMARY KEY,
  Firstname TEXT    NOT NULL,
  Lastname  TEXT    NOT NULL,
  VersionID INTEGER NOT NULL DEFAULT 0
);
```

Three data columns, plus `VersionID` for optimistic locking — Trysil supports it out of the box.

## The entity

Here is the class that maps to that table:

```pascal
{$WARN UNKNOWN_CUSTOM_ATTRIBUTE ERROR}

type
  [TTable('Persons')]
  [TSequence('PersonsID')]
  TPerson = class
  strict private
    [TPrimaryKey]
    [TColumn('ID')]
    FID: TTPrimaryKey;

    [TColumn('Firstname')]
    FFirstname: String;

    [TColumn('Lastname')]
    FLastname: String;

    [TVersionColumn]
    [TColumn('VersionID')]
    FVersionID: TTVersion;
  public
    property ID: TTPrimaryKey read FID;
    property Firstname: String read FFirstname write FFirstname;
    property Lastname: String read FLastname write FLastname;
    property VersionID: TTVersion read FVersionID;
  end;
```

Three things worth noticing:

- **No constructor, no inheritance, no registration code.** Trysil discovers the class through RTTI thanks to the attributes.
- **`{$WARN UNKNOWN_CUSTOM_ATTRIBUTE ERROR}`** turns misspelled attribute names into compile errors instead of silent runtime bugs. Always enable it in units that use Trysil attributes.
- **`ID` and `VersionID` are read-only** from the outside. The framework writes to them; your code doesn't.
- **`[TSequence]`** is ignored by SQLite (which uses `MAX(ROWID) + 1`), but the moment you point the same entity at PostgreSQL, Firebird, or SQL Server with a sequence object, the mapping is already complete. Declare it once, portable forever.

That's the whole mapping. No XML, no fluent configuration, no code generator.

## The connection

Before you can read or write anything, you need a connection and a context:

```pascal
program FirstEntity;

{$APPTYPE CONSOLE}

uses
  Trysil.Types,
  Trysil.Attributes,
  Trysil.Generics.Collections,
  Trysil.Data,
  Trysil.Data.FireDAC.SQLite,
  Trysil.Context;

var
  LConnection: TTConnection;
  LContext: TTContext;
begin
  ReportMemoryLeaksOnShutdown := True;

  TTSQLiteConnection.RegisterConnection('Main', 'persons.db');

  LConnection := TTSQLiteConnection.Create('Main');
  try
    LContext := TTContext.Create(LConnection);
    try
      // ... work goes here ...
    finally
      LContext.Free;
    end;
  finally
    LConnection.Free;
  end;
end.
```

Two steps: register a named FireDAC connection once (`RegisterConnection`), then create a `TTSQLiteConnection` pointing at that name. `TTContext` is the single entry point — every SELECT, INSERT, UPDATE, and DELETE goes through it.

## Read: SelectAll

```pascal
var
  LPersons: TTList<TPerson>;
  LPerson: TPerson;
begin
  LPersons := LContext.CreateEntityList<TPerson>();
  try
    LContext.SelectAll<TPerson>(LPersons);
    for LPerson in LPersons do
      Writeln(Format('%d: %s %s',
        [LPerson.ID, LPerson.Firstname, LPerson.Lastname]));
  finally
    LPersons.Free;
  end;
end;
```

Two notes:

- `SelectAll` is a **procedure**, not a function. You create the list, Trysil fills it.
- `LContext.CreateEntityList<T>()` returns a `TTList<T>` with ownership handled internally. When the identity map is active (the default), the list is non-owning — the context owns the entities. When it's off, the list owns them. Same code, both cases, no leaks.

## Create: Insert

```pascal
var
  LPerson: TPerson;
begin
  LPerson := LContext.CreateEntity<TPerson>();
  try
    LPerson.Firstname := 'David';
    LPerson.Lastname := 'Lastrucci';
    LContext.Insert<TPerson>(LPerson);
    Writeln(Format('Inserted with ID = %d', [LPerson.ID]));
  finally
    LContext.FreeEntity<TPerson>(LPerson);
  end;
end;
```

`CreateEntity<T>` creates a fresh instance. `FreeEntity<T>` is its counterpart: if the identity map is active it's a no-op (the context owns the entity), otherwise it frees it. You don't have to think about which case you're in — the context knows and does the right thing. Same principle as `CreateEntityList<T>` above: the ownership policy is decided once on the `TTContext`, not sprinkled through every call site.

## Update: Get, modify, Update

```pascal
var
  LPerson: TPerson;
begin
  if LContext.TryGet<TPerson>(1, LPerson) then
    try
      LPerson.Lastname := 'Lastrucci-Updated';
      LContext.Update<TPerson>(LPerson);
    finally
      LContext.FreeEntity<TPerson>(LPerson);
    end;
end;
```

`TryGet<T>(1, LPerson)` fetches the row with `ID = 1` through the identity map and returns `True` if found; the `out` parameter handles the "not found" case without null checks. Repeated calls in the same context return the same object. After `Update`, `VersionID` is incremented — if another process updated the row in the meantime, you get an `ETVersionException` and your change is safely rejected. That's optimistic locking in one attribute.

## Delete

```pascal
var
  LPerson: TPerson;
begin
  if LContext.TryGet<TPerson>(1, LPerson) then
    try
      LContext.Delete<TPerson>(LPerson);
    finally
      LContext.FreeEntity<TPerson>(LPerson);
    end;
end;
```

Same shape as Insert and Update. If the entity has a `[TDeletedAt]` column, `Delete` becomes a soft delete — a topic for another post.

## A note on memory

Ownership depends on one switch: **`TTContext.UseIdentityMap`**. When it's on (the default), the context owns every entity it hands you or creates. When it's off, your code owns them.

Two helpers on the context absorb the entire distinction:

| When you want | Call |
|---|---|
| A list to receive `Select` / `SelectAll` results | `LContext.CreateEntityList<T>()` |
| To dispose of a single entity after work | `LContext.FreeEntity<T>(LEntity)` |

Each one checks `UseIdentityMap` internally and does the right thing. You never write `if not LContext.UseIdentityMap then …` in your own code — that check lives in exactly one place, inside the framework. Flip the constructor flag and everything keeps working.

What you always free yourself: the `TTList<T>` returned by `CreateEntityList`, the `TTContext`, and the `TTConnection`.

## What's next

From here, the natural next steps are:

- **Filtering.** `TTFilterBuilder<T>` lets you build type-safe `WHERE` clauses fluently: `Where('Lastname').Equal('Smith').OrderByAsc('Firstname').Build`.
- **Relations.** The `[TRelation]` attribute and `TTLazy<T>` handle related entities on demand.
- **JSON and HTTP.** `TTJSonContext` serializes entities; `Trysil.Http` exposes them over REST.

Those will come in future posts. For now, if you have a table and a class, you have what you need to start.

Happy coding.
