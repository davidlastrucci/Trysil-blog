---
title: "Lazy loading: TTLazy<T> and TTLazyList<T>"
date: 2026-08-16 08:04:00 +0200
categories: [Tutorials]
tags: [lazy-loading, relations, tutorial]
---

A `TOrder` has a `CustomerID`. Sometimes, when the code holds an order, it also needs the `Customer`. Sometimes it doesn't.

A naïve ORM fetches the customer eagerly on every `Get<TOrder>`. That's simple but wasteful — you pay for a second query (or a bigger join) even when nobody asked. An aggressive ORM proxies the property: the customer object exists lazily, and the first access triggers the fetch. That's smart but expensive in Delphi, where there's no runtime proxy generation.

Trysil takes a middle path: two small generic types, `TTLazy<T>` and `TTLazyList<T>`, that you *embed* on the entity. They defer the fetch to first access, and — the part that matters for correct use — **the framework creates and frees them for you**. You declare the field; Trysil handles the rest.

## TTLazy<T>: the single related entity

The pattern for one-to-one / many-to-one:

```pascal
[TTable('Orders')]
TOrder = class
strict protected
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TColumn('CustomerID')]
  FCustomer: TTLazy<TCustomer>;

  [TColumn('Amount')]
  FAmount: Double;

  function GetCustomer: TCustomer;
  procedure SetCustomer(const AValue: TCustomer);
public
  property ID: TTPrimaryKey read FID;
  property Customer: TCustomer read GetCustomer write SetCustomer;
  property Amount: Double read FAmount write FAmount;
end;
```

Two details that look wrong at first glance and aren't:

- **There is no `FCustomerID: TTPrimaryKey` field.** The lazy *is* the foreign-key column. The `[TColumn('CustomerID')]` sits on the lazy field itself. Trysil reads the FK value from the lazy when serializing, and writes it back into the lazy when hydrating. One field, one attribute, one role.
- **There is no `constructor Create` and no `destructor Destroy`.** Trysil's RTTI code detects lazy fields during entity hydration (`TTRttiMember.CreateObject` in `Trysil.Rtti.pas`) and instantiates them for you, wiring each to the owning context and the column name from the attribute. Lifetime follows the owning entity — when the entity is freed, the lazy is freed with it.

Getter and setter are trivial:

```pascal
function TOrder.GetCustomer: TCustomer;
begin
  result := FCustomer.Entity;
end;

procedure TOrder.SetCustomer(const AValue: TCustomer);
begin
  FCustomer.Entity := AValue;
end;
```

Accessing `FCustomer.Entity` triggers the fetch the first time (an ordinary `Context.Get<TCustomer>(ID)` under the hood), and returns the cached object on subsequent calls. Assigning to `FCustomer.Entity := someCustomer` tells the lazy to point at a new object and automatically updates the FK value internally — no separate field to update, no chance of the FK and the cached entity getting out of sync.

This is the whole contract. No `Create`, no `Free`, no `ID := FCustomerID` dance. If any of those appear in your code, something is wrong — you've gone around the framework instead of through it.

## TTLazyList<T>: the related collection

The inverse relation — "all orders of this customer" — uses `TTLazyList<T>`:

```pascal
[TTable('Customers')]
TCustomer = class
strict protected
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TColumn('CompanyName')]
  FCompanyName: String;

  [TDetailColumn('ID', 'CustomerID')]
  FOrders: TTLazyList<TOrder>;

  function GetOrders: TTList<TOrder>;
public
  property ID: TTPrimaryKey read FID;
  property CompanyName: String read FCompanyName write FCompanyName;
  property Orders: TTList<TOrder> read GetOrders;
end;
```

```pascal
function TCustomer.GetOrders: TTList<TOrder>;
begin
  result := FOrders.List;
end;
```

Again: no constructor, no destructor, no manual anything.

The attribute this time is `[TDetailColumn('ID', 'CustomerID')]`. Two arguments:

- **First: the master's primary-key column** — `'ID'` on `Customers`.
- **Second: the detail's foreign-key column** — `'CustomerID'` on `Orders`.

The lazy builds a `WHERE Orders.CustomerID = :masterID` filter and calls `Context.Select<TOrder>` the first time you access `FOrders.List`. Subsequent accesses return the cached list.

## The N+1 trap

Every lazy-loading tutorial needs a warning sign here.

```pascal
for LOrder in LOrders do
  Writeln(LOrder.Customer.CompanyName);
```

Looks innocent. Runs one query to fetch the customer *for every order in the list*. A hundred orders → a hundred customer lookups. This is the N+1 problem in its textbook form, and it does not care that the iteration is happening inside Delphi's RTL.

The identity map softens this — if the same customer is referenced by ten orders, the context caches the first fetch and serves the other nine from memory (see the identity-map post, coming up next). But that only flattens it from N+1 to however many *distinct* customers you touched, which for many real datasets is still most of them.

Rule of thumb:

- If you're iterating a list and reading a related entity on every item, **don't use lazy loading.** Use `[TJoin]` and get everything in one SELECT. That's what `[TJoin]` is for.
- If you're reading a single entity and *maybe* need the related entity, use `TTLazy<T>`. You save a join when the related data isn't touched.
- If you're loading a parent to display its children on-screen later, `TTLazyList<T>` saves you from writing the child `Select` by hand.

`[TJoin]` and lazy loading are complementary. Neither replaces the other. The choice is about *access pattern*.

## When lazy is actually right

A good fit: the order detail screen. The user opens one order. You need the order, the customer, the shipping address, the billing address. Four `Get` calls, most of which run only if the user scrolls down to the billing section.

Another good fit: a CRUD form. The user loads one customer to edit. The form shows a "past orders" tab. Most users don't click that tab. If you always fetched orders, you'd be doing it for no reason 80% of the time. `TTLazyList<T>` defers the fetch to the moment the user actually clicks.

Bad fit: any "grid of parents with one column showing a field from the child". Use `[TJoin]`.

## Why the framework owns the lifecycle

Two reasons this matters, beyond "one less line of ceremony".

**Consistency with the identity map.** With the identity map on, the context owns every hydrated entity — including the ones a lazy points at. If you instantiated the lazy yourself, you'd need to know whether to free the cached entity at lazy-destruction time: *did this one come from the context or did I fetch it independently?* The framework-owned lifetime collapses that question — there's one source of truth, and the lazy and the context coordinate through it.

**Correctness of cache invalidation.** When the framework writes an updated entity back to the database, it needs to know which lazy fields carry which FK columns so it can synchronise them. The RTTI walk that hydrates lazies also indexes them for the write side. Manual instantiation would break that invariant — the framework would find a lazy with no registered role, and either crash or silently misbehave.

Neither problem exists in the declarative API. You say *"this is a lazy foreign key"*; Trysil does everything else.

## Validation interaction

The `[TRequired]` validation attribute (from the validation post) understands lazies. If a field is a `TObject` whose type starts with `TTLazy<` or `TTLazyList<`, the validator reads *through* the lazy — it checks the cached entity reference, not the wrapper — so `[TRequired]` on a lazy field does the expected thing. You don't need a separate attribute; the base validator handles the case internally.

## Closing

`TTLazy<T>` and `TTLazyList<T>` are fields you *declare* on your entity — with `[TColumn]` for single relations and `[TDetailColumn]` for collections. You never construct them, never free them, never manage their IDs. The framework does it all via RTTI during hydration.

The elegance is that the lazy field carries both the FK mapping *and* the navigation property. One declaration, one role. The user-facing surface is a trivial getter/setter, and the complexity lives where it belongs: inside the framework.

Next: the identity map — what it is, what it saves you, what it can hide.
