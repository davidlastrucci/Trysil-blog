---
title: "Why Trysil uses full-clone in TTSession"
date: 2026-04-23 19:30:00 +0200
categories: [Design Notes]
tags: [session, internals, generics, delphi]
---

The Unit of Work pattern is a classic. You hand an object a list of entities, it tracks their changes, and at commit time it figures out which fields changed on which entities and emits the minimal set of `INSERT` / `UPDATE` / `DELETE` statements. Hibernate does it. Entity Framework does it. Most mature ORMs do it.

Trysil does it too — `TTSession<T>` is exactly that. But its implementation is different from those other frameworks in one striking way: when you put an entity into a session, Trysil **clones it whole**. Not a lazy proxy, not a compiled interceptor, not a field-level diff object. A full, independent copy of the entity that sits next to the original until you call `ApplyChanges`.

This post is about **why that's the right answer for Delphi**, and why the "better" approaches used by other ORMs aren't available to us.

## How Hibernate and EF do it

When you read an entity from Hibernate or Entity Framework, what you actually get back is not your class — it's a **runtime-generated subclass** of your class. The ORM has intercepted every property setter and inserted bookkeeping: *"the `Lastname` field of entity with ID=42 was written."* At commit time, the ORM consults that dirty-field table and emits `UPDATE Persons SET Lastname = ? WHERE ID = 42`.

This is elegant. It's efficient: no shadow copies in memory. And it's mostly transparent to the developer — until you notice your `TPerson` is actually `TPerson_Proxy$$3f7a` in the debugger.

Everything about it depends on the runtime's ability to **synthesize a new class at execution time**. Java has `java.lang.reflect.Proxy` and CGLIB. .NET has `System.Reflection.Emit` and Castle DynamicProxy. Both platforms let you take a compiled class, generate a subclass that intercepts every virtual method, and hand that subclass back to the user as if it were the original.

## What Delphi doesn't have

Delphi has RTTI. It has generics. It has attributes. It has most of what modern OOP languages have.

It does **not** have runtime class generation. You cannot, from running Delphi code, produce a new `class` type that the rest of the program can instantiate. `TRttiType` is read-only. There is no `System.Reflection.Emit` equivalent. Property setters are compiled statically — intercepting them at runtime would require rewriting the binary.

This isn't a library gap. It's a language-level constraint.

So the dynamic-proxy approach is off the table. Not "hard". Not "awkward". Impossible.

## What's left

Given that constraint, two options remain for tracking dirty fields:

1. **Ask the developer to mark dirty fields manually** — provide a `SetDirty('Lastname')` API or similar bookkeeping. This works, but it defeats the point of a Unit of Work: the whole appeal of the pattern is that the framework notices changes for you. If you have to tell the framework what changed, you might as well call `Update` yourself.

2. **Keep a shadow copy of the original state, then diff at commit time.** When an entity enters the session, clone it. At `ApplyChanges`, compare live vs clone field by field — whatever changed, emit the appropriate SQL for those columns.

Option 2 is what `TTSession<T>` does. Every entity you give it is cloned through `TTContext.CloneEntity<T>`. The clones live inside the session. `ApplyChanges` walks both sets via RTTI and computes the delta.

## The code, roughly

```pascal
var
  LPersons: TTList<TPerson>;
  LSession: TTSession<TPerson>;
begin
  LPersons := LContext.CreateEntityList<TPerson>();
  try
    LContext.SelectAll<TPerson>(LPersons);
    LSession := LContext.CreateSession<TPerson>(LPersons);
    try
      // Modify freely — no framework calls needed
      LPersons[0].Lastname := 'Updated';
      LSession.ApplyChanges;
    finally
      LSession.Free;
    end;
  finally
    LPersons.Free;
  end;
end;
```

No `MarkDirty`, no `Attach`, no ceremony. The session knows what changed because it has a reference copy to compare against.

## The trade-off

Full-clone doubles the entity memory footprint for the duration of the session. For 100 rows that's noise. For 100,000 rows in-session it's real. Cloning isn't free either — copying an entity with RTTI costs more than flipping a flag.

What you get in return is **correctness that's verifiable by reading the code**. There's no generated proxy class to audit, no runtime intercept table to reason about. The shadow copy is just an object, and the diff is just a comparison. You can step through it in the debugger. It works with `strict private` fields, it works with `TTNullable<T>`, it works with every edge case of the Delphi type system because it relies only on features the compiler already provides.

For an ORM whose target use case is business applications with sessions in the hundreds of rows, that's the right budget.

## Closing

`TTSession<T>` using full-clone isn't a workaround. It isn't a "Delphi is limited, so we cope." It's the correct choice given the type system we have, and the only one that preserves the Unit of Work pattern's core promise: the developer writes regular code, the framework figures out the rest.

Every time I've been tempted to hand-roll dirty tracking with manual flags, I've ended up putting a flag in the wrong place and missing an update. The shadow copy doesn't have that failure mode.

Sometimes the simple implementation is the one the language lets you write.
