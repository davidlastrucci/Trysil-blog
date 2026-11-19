---
title: "Trysil.JSon: serializing entities without boilerplate"
date: 2026-11-01 07:44:00 +0100
categories: [Tutorials]
tags: [json, serialization, trysil-json, tutorial]
---

This post is about the `Trysil.JSon` module: turning entities into JSON, parsing JSON back into entities, and the one record that decides how much of the graph goes along.

The module exists because almost every real Trysil application eventually grows an HTTP layer, and every HTTP layer needs JSON. Re-declaring how every entity serializes would be the same boilerplate Trysil's ORM layer was designed to eliminate.

## TTJSonContext

The main type is `TTJSonContext`, which extends `TTContext`. You construct it exactly like a regular context:

```pascal
uses
  Trysil.JSon.Context;

LJSonContext := TTJSonContext.Create(LConnection);
```

From the outside, it behaves like any other context - `Select`, `Insert`, `Get`, `CreateEntity` all work. On top of that, it adds the JSON methods:

| Method | What it does |
|---|---|
| `EntityToJSon<T>(entity, config)` | One entity -> JSON string |
| `EntityToJSonObject<T>(entity, config)` | One entity -> `TJSonObject` |
| `ListToJSon<T>(list, config)` | A list -> JSON array string |
| `ListToJSonArray<T>(list, config)` | A list -> `TJSonArray` |
| `EntityFromJSon<T>(json)` | JSON string -> new entity |
| `EntityFromJSonObject<T>(jsonValue)` | `TJSonObject` -> new entity |
| `ListFromJSon<T>(json, list)` | JSON array string -> fill a list |
| `ListFromJSonArray<T>(array, list)` | `TJSonArray` -> fill a list |
| `DatasetToJSon(dataset)` | Any `TDataSet` -> JSON |
| `MetadataToJSon<T>()` | The mapped members of an entity, with their names and types |

The `...Object` and `...Array` variants give you `TJSonValue` descendants when you want to compose them into a bigger JSON payload. The string variants are convenient when you're about to send the result down an HTTP response.

## A first round trip

Using the `TOrder` entity from earlier posts:

```pascal
uses
  Trysil.JSon.Context,
  Trysil.JSon.Types;   // for TTJSonSerializerConfig

var
  LConfig: TTJSonSerializerConfig;
  LOrder: TOrder;
  LJson: String;
begin
  LConfig := TTJSonSerializerConfig.EntityOnly;

  LOrder := LJSonContext.Get<TOrder>(42);
  try
    LJson := LJSonContext.EntityToJSon<TOrder>(LOrder, LConfig);
    Writeln(LJson);
  finally
    LJSonContext.FreeEntity<TOrder>(LOrder);
  end;
end;
```

Output:

```json
{
  "id": 42,
  "customerID": 7,
  "amount": 1499.50,
  "createdAt": "2026-04-01T09:33:12.000Z",
  "createdBy": "alice",
  "updatedAt": null,
  "updatedBy": "",
  "deletedAt": null,
  "deletedBy": "",
  "versionID": 3
}
```

Every `[TColumn]` member is a JSON property. Nullables that are null become `null`. Timestamps are ISO-8601. No configuration needed for the common case.

The key comes from the **member** name, not from the column name: the `F` prefix is dropped when what follows it is uppercase, and the first letter is lowercased. So `FCustomerID` is `customerID` and `FAmount` is `amount`, while a member genuinely named `FatturaID` - no prefix, an `F` that belongs to the word - keeps it and comes out as `fatturaID`. The column behind the member is never named in the payload: rename a column and the JSON does not move.

## TTJSonSerializerConfig

`TTJSonSerializerConfig` is a *record*, not a class, so a variable of that type that nobody assigned holds whatever was in memory. Always initialize it, either with `Create(AMaxLevels, ADetails)` or with one of the presets:

| Preset | Equivalent | Use |
|---|---|---|
| `Default` | `Create(-1, False)` | the whole graph of references, no detail lists |
| `WithDetails` | `Create(1, True)` | one entity with its references and its detail lists |
| `WithRelations` | `Create(1, False)` | one level of references, no detail lists |
| `EntityOnly` | `Create(0, False)` | the entity alone, references as their ids |

The two arguments:

- **`MaxLevels` (first arg).** How deep to follow related entities. `0` does not follow them at all: a `TTLazy<T>` is written as its foreign key id, **without a query**. `1` resolves the first level, so each reference becomes a nested object. `-1` means *no limit*, and follows the graph as far as it goes, stopped only by the cycle guard.
- **`Details` (second arg).** Whether to serialize the `[TDetailColumn]` collections - the master-detail lists hanging off the entity. `False` leaves them out entirely.

The gate that stops the queries is the same one that lets them through, so the number is not a formatting preference, it is a query budget:

- **A single entity with its references one level deep:** `WithRelations`, or `WithDetails` to bring its detail lists along.
- **A list:** with `MaxLevels = 1` every reference of every row is resolved, which is one query per relation per row: a list of 50 rows with 4 relations costs **2 queries** with `EntityOnly` and **202** with `WithRelations`. Often it is exactly what you want - a grid of invoices that shows the customer's name next to each row needs that reference resolved - so it is a price to pay knowingly, not a mistake to avoid.
- **A dump where you really do want the whole graph:** `Create(-1, True)`. Know what you are asking for.

The shape of the JSON should differ between use cases, and the HTTP module's generic reader and writer (`TTHttpEntityReader<T>` and `TTHttpEntityWriter<T>`, next posts) have a default preset per method, each overridable with an overload that takes your own config.

## Serializing a list

```pascal
var
  LList: TTList<TOrder>;
begin
  LList := LJSonContext.CreateEntityList<TOrder>();
  try
    LJSonContext.SelectAll<TOrder>(LList);
    LJson := LJSonContext.ListToJSon<TOrder>(LList, LConfig);
  finally
    LList.Free;
  end;
end;
```

Output is a JSON array of the same shape as a single entity:

```json
[
  { "id": 42, "customerID": 7, ... },
  { "id": 43, "customerID": 9, ... },
  ...
]
```

If you want to wrap the array in a count + data envelope (the shape most pagination UIs want), use `ListToJSonArray` and compose:

```pascal
var
  LEnvelope: TJSonObject;
  LArray: TJSonArray;
begin
  LEnvelope := TJSonObject.Create;
  try
    LEnvelope.AddPair('count',
      TJSonNumber.Create(LJSonContext.SelectCount<TOrder>(TTFilter.Empty())));
    LArray := LJSonContext.ListToJSonArray<TOrder>(LList, LConfig);
    try
      LEnvelope.AddPair('data', LArray);
    except
      LArray.Free;
      raise;
    end;
    LJson := LEnvelope.ToJSon;
  finally
    LEnvelope.Free;
  end;
end;
```

This is exactly the pattern the HTTP module uses for paged endpoints.

## Deserialization

Going the other way, the symmetry is clean:

```pascal
LOrder := LJSonContext.EntityFromJSon<TOrder>(LIncomingJson);
try
  LJSonContext.Update<TOrder>(LOrder);
finally
  LJSonContext.FreeEntity<TOrder>(LOrder);
end;
```

`EntityFromJSon<T>` creates a new instance, populates every mapped field from the JSON object, and hands it back. You own the resulting object.

A few rules the deserializer enforces:

- **Keys are matched exactly, case included.** They are the camelCase names the serializer writes, so `"amount"` is read and `"Amount"` is not.
- **Unknown JSON fields are ignored.** If the payload has `"discount": 0.1` and the entity doesn't, nothing happens - no exception. The same goes for a key with the wrong case, which is an unknown key.
- **Missing fields:** a plain field the payload omits keeps its default value, while a missing `TTNullable` is set to null. That difference matters when the JSON is read into an entity that already has values.
- **Type mismatches throw.** `"amount": "not a number"` raises `ETJSonException`.
- **Identity map is off.** `TTJSonContext` forbids the identity map - if you try to create it with `AUseIdentityMap := True`, it raises `ETJSonException`. Serialization is typically many-entities-once; the map would add lifetime questions the module deliberately avoids.

## Round-tripping: the insert flow

The typical incoming-insert pattern:

```pascal
LOrder := LHttpContext.EntityFromJSonObject<TOrder>(LRequestJSon);
try
  if LHttpContext.GetID<TOrder>(LOrder) <= 0 then
    LHttpContext.SetSequenceID<TOrder>(LOrder);
  LHttpContext.Insert<TOrder>(LOrder);
  LResponse := LHttpContext.EntityToJSon<TOrder>(LOrder, LGetConfig);
finally
  LHttpContext.FreeEntity<TOrder>(LOrder);
end;
```

Four steps: parse incoming JSON, set the sequence ID if the client sent none, insert, serialize the result (including the new ID and any server-side computed fields like `CreatedAt`). `GetID` and `SetSequenceID` live on `TTHttpContext`, the descendant of `TTJSonContext` that the HTTP module uses for one request. You rarely write this by hand: `TTHttpEntityWriter<T>.Insert` is exactly this flow.

## Datasets, too

`DatasetToJSon(dataset)` is an escape hatch for any `TDataSet` that isn't an entity - say, the result of a `TTContext.CreateDataset` call against a raw SQL. You lose the attribute-driven mapping (the dataset's column names show up verbatim), but you get JSON without threading the data through an entity class first.

## Closing

`TTJSonContext` gives you JSON in and JSON out with one method each, for single entities and for lists. `TTJSonSerializerConfig` is the only configuration surface, and it only matters when your needs deviate from the defaults (how deep to follow references; whether to include the detail lists). Identity-map support is deliberately absent - the tradeoff for a module that treats entities as values, not references.

Next: the HTTP module - a REST API from zero, with the controllers and the routing wired by attributes.
