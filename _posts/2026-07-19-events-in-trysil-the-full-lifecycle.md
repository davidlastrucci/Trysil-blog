---
title: "Events in Trysil: the full lifecycle"
date: 2026-07-19 07:52:00 +0200
categories: [Deep Dives]
tags: [events, lifecycle, attributes]
---

The previous post ended with a promise: *after* hooks exist, we'll come back to them. This is that post.

Trysil has two complementary mechanisms for reacting to persistence operations. One is the method-hook approach from the validation post — `[TBeforeInsert]`, `[TBeforeUpdate]` and friends, lightweight in-class procedures. The other is the *event class* mechanism, which you reach for when the reaction outgrows the entity: audit logs, outbound notifications, cache invalidation, anything that deserves its own unit.

Both are driven by attributes. Neither needs subclassing. The resolver threads them all into one pipeline.

## The two styles, side by side

Method hook — the short version:

```pascal
[TBeforeInsert]
procedure CheckTotals;
```

Event class — the longer version:

```pascal
[TInsertEvent(TInvoiceAuditEvent)]
TInvoice = class
  // fields
end;
```

The method hook lives *on the entity*, sees `Self`, runs inline. The event class lives *somewhere else*, receives the entity by reference, and runs as a standalone object that the framework instantiates, calls, and frees.

Pick the one whose shape matches the concern. Cross-field validation: method hook. Writing a row to an audit table: event class. The distinction is *does this logic belong to the entity or beside it*.

## Method hooks, in full

The six method-level attributes:

| Attribute | Fires |
|---|---|
| `[TBeforeInsert]` | Before `Insert` — abort by raising |
| `[TAfterInsert]` | After `Insert` commits |
| `[TBeforeUpdate]` | Before `Update` — abort by raising |
| `[TAfterUpdate]` | After `Update` commits |
| `[TBeforeDelete]` | Before `Delete` — abort by raising |
| `[TAfterDelete]` | After `Delete` commits |

A method can carry any combination:

```pascal
[TBeforeInsert]
[TBeforeUpdate]
procedure ValidateTotals;

[TAfterInsert]
[TAfterUpdate]
procedure QueueReindex;
```

The resolver invokes them in attribute-declaration order. No priorities, no explicit sequencing — if two hooks must run in a specific order, put them in the same method.

## Event classes, in full

The six *class-level* attributes:

```pascal
[TInsertEvent(TSomeInsertEventClass)]
[TUpdateEvent(TSomeUpdateEventClass)]
[TDeleteEvent(TSomeDeleteEventClass)]
TMyEntity = class
end;
```

Each attribute names an event class. The resolver instantiates it around the corresponding operation, calls `DoBefore` before the SQL, `DoAfter` after the commit, and frees the instance. One event class per operation type, per entity.

The event class extends `TTEvent`:

```pascal
TTEvent = class abstract
public
  procedure DoBefore; virtual;
  procedure DoAfter; virtual;
end;
```

Both methods have empty default implementations. You override whichever you need. If your event class only overrides `DoAfter`, `DoBefore` is a no-op (and vice versa).

## A concrete event class

Say you want to stream every customer change to an internal message bus. The method-hook approach would bloat `TCustomer.pas` with bus-plumbing code. The event-class approach keeps the entity clean:

```pascal
unit Audit.Customer.Events;

interface

uses
  Trysil.Context,
  Trysil.Events.Abstract,
  App.MessageBus;

type
  TCustomerInsertEvent = class(TTEvent)
  strict private
    FContext: TTContext;
    FEntity: TCustomer;
  public
    constructor Create(
      const AContext: TTContext; const AEntity: TCustomer);

    procedure DoAfter; override;
  end;

implementation

constructor TCustomerInsertEvent.Create(
  const AContext: TTContext; const AEntity: TCustomer);
begin
  inherited Create;
  FContext := AContext;
  FEntity := AEntity;
end;

procedure TCustomerInsertEvent.DoAfter;
begin
  MessageBus.Publish('customers.created', TCustomerPayload.From(FEntity));
end;

end.
```

The constructor signature is load-bearing: `TTEventFactory` (`Trysil.Events.Factory.pas`) finds your class's constructor via RTTI and expects exactly two parameters — a context object and the entity. If your constructor has a different shape, the factory raises `ETException` at the first write operation. The two-parameter form is the contract.

Wire it up on the entity:

```pascal
uses
  Trysil.Events.Attributes,
  Audit.Customer.Events;

[TTable('Customers')]
[TInsertEvent(TCustomerInsertEvent)]
TCustomer = class
  // fields
end;
```

That's it. Every successful `Insert<TCustomer>` fires `DoAfter` after the transaction commits. Every failed insert never fires — the factory builds the event only if the resolver decides to proceed, and `DoAfter` only runs if the operation actually succeeded.

## How the resolver assembles the pipeline

Under the hood, the resolver — for any write operation — builds a small pipeline:

1. Read attributes from the entity class. Collect method-hook attributes, collect event-class attributes.
2. Instantiate any event class via the event factory (`Trysil.Events.Factory.pas`).
3. Run all `Before*` method hooks. Run `DoBefore` on the event instance. Any exception aborts and propagates.
4. Apply change-tracking attributes (`[TCreatedAt]`, `[TUpdatedAt]`, `[TDeletedAt]` and their `*By` companions).
5. Execute the SQL command inside the current transaction.
6. On success, run `DoAfter` on the event instance, then all `After*` method hooks.
7. Free the event instance.

The factory is the part that turns a class reference (`TCustomerInsertEvent`) into an instance. It calls the class's constructor passing the entity — which is why event classes need a constructor that takes the entity type. The constructor above is the standard shape.

## Before vs. after, for writes

The distinction matters more than it looks:

- **Before** runs *inside* the transaction, before the SQL. Raising an exception from a Before hook rolls back the whole operation. Use it for validation and for modifying the entity before it's written.
- **After** runs *after* the commit. Raising an exception here does *not* roll back — the SQL is already committed. The exception propagates to the caller, and any later After hooks in the same operation are skipped.

Concretely: a Before hook can stop the save. An After hook can only react. If your side-effect must be atomic with the save — say, writing to an audit table on the same database — put that write in `DoBefore` and let the framework's transaction protect both. If the side-effect is external (message bus, HTTP call, cache eviction), `DoAfter` is the right place because it runs after the database is committed and the external world can safely see the new state.

## Where events beat hooks

Three situations tip the scale toward event classes:

1. **Reusability across entities.** A generic "emit-to-audit-log" event can be written once and attached to a dozen entities. The method-hook version would need copying and pasting into each entity class.
2. **Entity cleanliness.** Business entities should describe their domain. A customer class full of message-bus code is noise in a code review.
3. **Testability.** An event class is an ordinary object with `DoBefore` / `DoAfter` methods. You can unit-test it by instantiating it with a stub entity and calling the method directly — no context, no connection, no database.

When none of those matter, the method hook is shorter and fine.

## Where hooks beat events

Three situations tip back to method hooks:

1. **Cross-field validation.** Any rule that needs `Self.*` from the entity is at home as an instance method.
2. **One-off rules.** A single rule on a single entity doesn't need the ceremony of a separate class.
3. **The rule doesn't outlive the entity.** If removing the entity class would also remove the rule, the rule probably belongs inside the entity class.

## Closing

Trysil gives you two event mechanisms because two shapes of problem show up. Method hooks handle the entity-local, domain-sized logic: "this field must match that sum". Event classes handle the cross-cutting, infrastructure-sized logic: "tell the bus about this, write the audit, evict the cache". Attributes wire both without demanding a class hierarchy, and the resolver runs them in a predictable Before/After pipeline around the transaction.

Next: optimistic locking with `TTVersion` — the one column that saves you from the race condition you didn't notice you had.
