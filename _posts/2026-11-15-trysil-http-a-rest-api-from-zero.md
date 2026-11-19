---
title: "Trysil.Http: a REST API from zero"
date: 2026-11-15 08:36:00 +0100
categories: [Tutorials]
tags: [rest, http, trysil-http, tutorial]
---

This post starts a four-post mini-series on `Trysil.Http` - the REST module that wraps the ORM in a conventional HTTP API layer.

The pattern is deliberate: you write entities, you attach attribute-annotated controllers, and you get working endpoints. Routing, serialization, filters, authentication, and CORS are all pluggable, all attribute-driven, and all built on the same conventions the rest of Trysil uses.

We'll walk through a minimal API exposing one entity, one controller, one endpoint. The next three posts expand into multi-tenant, authentication, and production concerns.

## The setup

Two units, beyond the entities: an application context, and a controller.

The context is a plain class that owns a connection and a `TTHttpContext`, with a **parameterless constructor**. That last part is a requirement, not a style: the server builds one of these per request and frees it when the response is written, so it has to be able to construct it with no arguments.

```pascal
unit App.Context;

interface

uses
  Trysil.Data,
  Trysil.Data.FireDAC.SQLite,
  Trysil.Http.Context;

type
  TAppContext = class
  strict private
    FConnection: TTConnection;
    FContext: TTHttpContext;
  public
    constructor Create;
    destructor Destroy; override;

    property Context: TTHttpContext read FContext;
  end;

implementation

constructor TAppContext.Create;
begin
  inherited Create;
  FConnection := TTSQLiteConnection.Create('main');
  FContext := TTHttpContext.Create(FConnection);
end;

destructor TAppContext.Destroy;
begin
  FContext.Free;
  FConnection.Free;
  inherited Destroy;
end;

end.
```

`TTHttpContext` rather than `TTJSonContext` because it is the one that knows it lives for a single request. It carries the JSON layer exactly the same way, and it refuses `Save<T>`: `Save` decides between an insert and an update from what the context created and has not written yet, and a context built and destroyed inside one request has no such history. Whether a request is a create or an update is in the request, so a controller calls `Insert<T>` or `Update<T>` and says which.

Put whatever else is per-request in there beside them - the decoded JWT payload, the tenant, a correlation id. Controllers reach it through this class.

## The controller

A controller is a class that extends `TTHttpController<TContext>`, where the generic parameter is your context class:

```pascal
unit App.Controller.Orders;

{$WARN UNKNOWN_CUSTOM_ATTRIBUTE ERROR}

interface

uses
  System.SysUtils,
  System.JSon,

  Trysil.Types,
  Trysil.Filter,
  Trysil.Generics.Collections,
  Trysil.JSon.Types,
  Trysil.Http.Exceptions,
  Trysil.Http.Attributes,
  Trysil.Http.Controller,

  App.Context,
  App.Model.Order;

type
  [TUri('/orders')]
  TOrderController = class(TTHttpController<TAppContext>)
  strict private
    FConfig: TTJSonSerializerConfig;
  public
    procedure AfterConstruction; override;

    [TGet]
    procedure SelectAll;

    [TGet('/?')]
    procedure GetByID(const AID: TTPrimaryKey);

    [TPost]
    procedure Insert;

    [TPut]
    procedure Update;

    [TDelete('/?/?')]
    procedure Delete(const AID: TTPrimaryKey; const AVersion: TTVersion);
  end;

implementation

procedure TOrderController.AfterConstruction;
begin
  inherited AfterConstruction;
  FConfig := TTJSonSerializerConfig.WithRelations;
end;

procedure TOrderController.SelectAll;
var
  LList: TTList<TOrder>;
begin
  LList := FContext.Context.CreateEntityList<TOrder>;
  try
    FContext.Context.SelectAll<TOrder>(LList);
    FResponse.Content :=
      FContext.Context.ListToJSon<TOrder>(LList, FConfig);
  finally
    LList.Free;
  end;
end;

procedure TOrderController.GetByID(const AID: TTPrimaryKey);
var
  LOrder: TOrder;
begin
  if not FContext.Context.TryGet<TOrder>(AID, LOrder) then
    raise ETHttpNotFound.CreateFmt('Order %d not found', [AID]);
  try
    FResponse.Content :=
      FContext.Context.EntityToJSon<TOrder>(LOrder, FConfig);
  finally
    FContext.Context.FreeEntity<TOrder>(LOrder);
  end;
end;

procedure TOrderController.Insert;
var
  LOrder: TOrder;
begin
  LOrder := FContext.Context.EntityFromJSonObject<TOrder>(FRequest.JSonContent);
  try
    if FContext.Context.GetID<TOrder>(LOrder) <= 0 then
      FContext.Context.SetSequenceID<TOrder>(LOrder);
    FContext.Context.Insert<TOrder>(LOrder);
    FResponse.Content := FContext.Context.EntityToJSon<TOrder>(LOrder, FConfig);
  finally
    FContext.Context.FreeEntity<TOrder>(LOrder);
  end;
end;

procedure TOrderController.Update;
var
  LOrder: TOrder;
begin
  LOrder := FContext.Context.EntityFromJSonObject<TOrder>(FRequest.JSonContent);
  try
    FContext.Context.Update<TOrder>(LOrder);
    FResponse.Content := FContext.Context.EntityToJSon<TOrder>(LOrder, FConfig);
  finally
    FContext.Context.FreeEntity<TOrder>(LOrder);
  end;
end;

procedure TOrderController.Delete(
  const AID: TTPrimaryKey; const AVersion: TTVersion);
begin
  FContext.Context.Delete<TOrder>(AID, AVersion);
end;

end.
```

Five endpoints: list, get by id, insert, update, delete. Under 80 lines. Most of it is the boilerplate of hydrating JSON in, hydrating JSON out, and managing the entity's lifetime.

## The attributes that drive it

| Attribute | Effect |
|---|---|
| `[TUri('/orders')]` | Class-level prefix - every route on this controller starts here |
| `[TGet]` | GET with no additional path segment - matches `/orders` |
| `[TGet('/?')]` | GET with one path parameter - matches `/orders/42` |
| `[TPost]` | POST with no additional segment |
| `[TPut]` | PUT |
| `[TDelete('/?/?')]` | DELETE with two path parameters - matches `/orders/42/7` |

The `?` is a placeholder for a positional parameter. The method signature determines the type - `TTPrimaryKey` for an int, `String` for a string, and so on. The router maps `/orders/42/7` onto `GetByID(42, 7)` without any manual parsing.

## Starting the server

`TTHttpServer<C>` is the entry point, generic over the context class. You set the properties, register the controllers, and start it:

```pascal
program API;

uses
  Trysil.Http,

  App.Context,
  App.Controller.Orders;

var
  LServer: TTHttpServer<TAppContext>;
begin
  LServer := TTHttpServer<TAppContext>.Create;
  try
    LServer.Port := 8080;
    LServer.BaseUri := '/api';
    LServer.AllowAnonymous := True;

    LServer.RegisterController<TOrderController>();

    LServer.Start;
    Writeln('Listening on http://localhost:8080/api');
    Readln;
  finally
    LServer.Free;
  end;
end.
```

Three things in there are worth a sentence each.

**`AllowAnonymous := True`.** Without it this program does not start, and the refusal is the point. A route with no `[TAuthorizationType]` requires authentication *by default* - the safe direction, and the opposite of the usual accident - so a server with no authentication class registered refuses to start rather than serving everything openly. Saying `AllowAnonymous := True` is you declaring that this really is a public API. We add authentication in a later post and this line goes away.

**`BaseUri`** is a path prefix glued in front of every route, not an address. The address comes from `Port` and the bindings, and a value carrying a scheme is refused for exactly that reason.

**`LServer.Free`** is enough. `Destroy` calls `Stop`, which waits for the request threads before returning.

## One context per request

There is no factory function to write. `TTHttpServer<C>` is declared `C: class, constructor`, so the server constructs a `TAppContext` for each request and frees it when the response is written. That is why the constructor had to take no arguments.

It also settles a question the identity-map post raised: **context-scoped means request-scoped**. Two requests never share entities, never share a transaction, and never share an identity map, and none of that depends on you remembering to make it so.

## Try it

With the listener running, a handful of curl calls:

```bash
# List all orders
curl http://localhost:8080/api/orders

# Get one by ID
curl http://localhost:8080/api/orders/42

# Create
curl -X POST http://localhost:8080/api/orders \
     -H 'Content-Type: application/json' \
     -d '{"customerID": 7, "amount": 99.0}'

# Update
curl -X PUT http://localhost:8080/api/orders \
     -H 'Content-Type: application/json' \
     -d '{"id": 42, "versionID": 3, "amount": 150.0}'

# Delete (ID + VersionID for optimistic locking)
curl -X DELETE http://localhost:8080/api/orders/42/3
```

The delete carries the version explicitly because the URL is the only transport - no body. The controller signature `Delete(AID, AVersion)` plus the `[TDelete('/?/?')]` template wire it up.

## Generic CRUD: TTHttpEntityReader and TTHttpEntityWriter

Most APIs have the same endpoints for every entity, and writing the controller above once per entity would be copy-paste with the type name changed. `Trysil.Http.Entity` holds the generic part:

- **`TTHttpEntityReader<T>`** - `Get`, `Find`, `SelectAll`, `Select` (with a JSON filter, paging and sorting) and `Metadata`.
- **`TTHttpEntityWriter<T>`** - `Insert`, `Update`, `Delete` and `CreateNew`.

They are not controllers. They borrow the `TTHttpContext` you pass to the constructor, take the body as a `TJSonValue` and return JSON as a string, so routes, areas and authentication stay where they belong, in your controller:

```pascal
procedure TOrderController.GetByID(const AID: TTPrimaryKey);
var
  LReader: TTHttpEntityReader<TOrder>;
begin
  LReader := TTHttpEntityReader<TOrder>.Create(FContext.Context);
  try
    FResponse.Content := LReader.Get(AID);
  finally
    LReader.Free;
  end;
end;
```

`Get` and `Find` raise `ETHttpNotFound` for an id that is not there, each method has a sensible serializer preset and an overload that takes your own, and the hooks you would otherwise write by hand - an extra filter, something to do before serializing, before and after each write - are events you assign.

The usual shape is a generic base controller per project, `TReadOnlyController<T>` and `TReadWriteController<T>`, that declares the routes once and delegates each one in a line. Every entity then gets a one-line subclass:

```pascal
[TUri('/orders')]
TOrderController = class(TReadWriteController<TOrder>);
```

A read-only resource simply never creates the writer.

## Error handling

The listener turns exceptions into status codes:

| Exception | HTTP status |
|---|---|
| `ETHttpException` and its descendants (`ETHttpNotFound`, `ETHttpBadRequest`, `ETHttpForbidden`, ...) | the status the class carries |
| `ETConcurrentUpdateException` - a stale version on update or delete | 409 Conflict |
| `ETValidationException` | 422 Unprocessable Content, with the error list |
| `ETJSonException` - a body that cannot be read | 400 Bad Request |
| Anything else | 500, with a fixed body; the detail goes to the log |

Raise the `ETHttp*` classes from your controller when the default mapping is not what you want. The 500 body is fixed on purpose: the class and message of an unexpected exception say too much about the inside of a server to hand them to whoever called it.

## Closing

`Trysil.Http` is a thin, attribute-driven routing layer on top of `TTJSonContext`. You get the five CRUD endpoints per entity in fewer lines than most manual REST implementations, and the wiring is explicit enough that "how does a request become a method call" has a short, readable answer.

Next: multi-tenant REST - one API, many databases, zero cross-tenant leakage.
