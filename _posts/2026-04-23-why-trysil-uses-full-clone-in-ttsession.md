---
title: "Why Trysil uses full-clone in TTSession"
date: 2026-04-23 19:30:00 +0200
categories: [Design Notes]
tags: [session, internals, generics, delphi]
---

The Unit of Work pattern is a classic. You hand an object a list of entities, you mutate them, and at commit time you get one transaction that reconciles memory with the database. Hibernate does it. Entity Framework does it. Most mature ORMs do it.

Trysil does it too — `TTSession<T>` is exactly that. But if you squint at its API, two things stand out: when you put an entity into a session, Trysil **clones it whole**, and when you want a mutation to actually be persisted, you have to call `LSession.Update(clone)` explicitly. No automatic detection. No proxied setter intercepts. A full copy of the entity sits next to the original, and you drive state transitions with API calls.

This post is about **why those two choices are the right answer for Delphi**.

## How Hibernate and EF do it

When you read an entity from Hibernate or Entity Framework, what you actually get back is not your class — it's a **runtime-generated subclass** of your class. The ORM has intercepted every property setter and inserted bookkeeping: *"the `Lastname` field of entity with ID=42 was written."* At commit time, the ORM consults that dirty-field table and emits `UPDATE Persons SET Lastname = ? WHERE ID = 42`.

This is elegant. It's efficient: no shadow copies in memory. And it's mostly transparent to the developer — until you notice your `TPerson` is actually `TPerson_Proxy$$3f7a` in the debugger.

Everything about it depends on the runtime's ability to **synthesize a new class at execution time**. Java has `java.lang.reflect.Proxy` and CGLIB. .NET has `System.Reflection.Emit` and Castle DynamicProxy. Both platforms let you take a compiled class, generate a subclass that intercepts every virtual method, and hand that subclass back to the user as if it were the original.

## What Delphi doesn't have

Delphi has RTTI. It has generics. It has attributes. It has most of what modern OOP languages have.

It does **not** have runtime class generation. You cannot, from running Delphi code, produce a new `class` type that the rest of the program can instantiate. `TRttiType` is read-only. There is no `System.Reflection.Emit` equivalent. Property setters are compiled statically — intercepting them at runtime would require rewriting the binary.

This isn't a library gap. It's a language-level constraint.

So the dynamic-proxy approach is off the table. Not "hard". Not "awkward". Impossible. Whatever shape a Unit of Work takes in Delphi, it cannot rely on intercepting writes to fields.

## What's left: two decisions, not one

Strip away the proxy magic and a UoW implementation is really two separable decisions — decisions that other ORMs conflate because proxies let them solve both for free.

**1. How does the session know which entities to write back?**

Without setter interception, two practical options remain:

- **Snapshot-and-diff.** Clone every entity on entry, walk clone vs. current at commit time, and treat "anything differs" as dirty. Opaque to the developer — mutate freely, `ApplyChanges` figures out the rest. Cost: a full RTTI comparison pass on every `ApplyChanges`, paid even when the caller already knows exactly what changed, plus a set of edge cases the comparator has to handle (float equality, `TTNullable<T>` null-vs-value, reference-typed fields).

- **Explicit state transitions.** The caller tells the session what's happening: `LSession.Update(clone)`, `LSession.Insert(entity)`, `LSession.Delete(clone)`. No diff, no inference — the session tracks a state per entity (`Original` / `Inserted` / `Updated` / `Deleted`) and walks those states at commit time.

Trysil picks the second. It trades a small amount of explicitness for zero magic: no RTTI diff pass, no failure modes where "I changed a field and nothing happened because of some equality edge case." If you called `Update`, it updates. If you didn't, it doesn't. The semantics are literally what the code says.

A direct consequence: `LSession.Update(clone)` emits an `UPDATE` even if you modified nothing. The session doesn't second-guess you.

**2. Do you mutate the original entity or a copy?**

Orthogonal to the first. You could do explicit state transitions while mutating the originals directly — no clone needed. You could do snapshot-and-diff while holding a parallel list of snapshots instead of cloning into a workspace. The two decisions compose four ways.

Trysil clones. Not for the diff (there is no diff), but for **isolation**:

- The originals are untouched until `ApplyChanges` runs. If it never runs — an exception, an abort, a form closed without saving — the originals are exactly as they came out of the database.
- `LSession.GetOriginalEntity(clone)` returns the pre-mutation snapshot, which makes undo, custom field-level diffs, and audit trails one-liners for the caller.
- If the same entity is referenced from elsewhere in the application (a cache, a data module, a bound grid), the session's in-flight edits don't leak into it. You avoid the class of bug where one form starts showing half-edited state because another form is mid-session.

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
      LSession.Entities[0].Lastname := 'Updated';
      LSession.Update(LSession.Entities[0]);

      LSession.ApplyChanges;
    finally
      LSession.Free;
    end;
  finally
    LPersons.Free;
  end;
end;
```

`LSession.Entities[0]` is the clone, not the original in `LPersons`. Mutate the clone, tell the session it's `Update`-state, apply. The originals in `LPersons` stay pristine until `ApplyChanges` returns.

## The trade-off

Full-clone doubles the entity memory footprint for the duration of the session. For 100 rows that's noise. For 100,000 rows in-session it's real. Cloning isn't free either — copying an entity with RTTI costs more than holding a shared reference.

What you get in return is **predictability you can read in the code**. No proxy class to audit, no diff pass with edge cases to reason about, no "why didn't my change stick?" debugging sessions. The rule is simple: you called `Update`, it updates. You didn't, it doesn't. The shadow copy is just an object and the state dictionary is just a dictionary.

For an ORM whose target use case is business applications with sessions in the hundreds of rows, that's the right budget.

## Closing

`TTSession<T>` isn't a workaround for missing dynamic proxies. It isn't "Delphi is limited, so we cope." It's a different answer to the same problem, built on features the language actually provides: explicit state transitions instead of inferred ones, and clones for isolation instead of diffs.

The result reads like regular Delphi — a list, some clones, three methods (`Insert`, `Update`, `Delete`), one `ApplyChanges`. No surprises, no hidden passes, no class-level magic.

Sometimes the simple implementation is the one the language lets you write.
