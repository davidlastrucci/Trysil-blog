---
title: "The Identity Map: what it gives you, what it hides"
date: 2026-08-30 07:38:00 +0200
categories: [Design Notes]
tags: [identity-map, internals, performance]
---

Here's a pattern that shows up in almost every ORM-backed codebase, written by people who don't realize it:

```pascal
LOrder := LContext.Get<TOrder>(42);
// ... do something ...
LOtherOrder := LContext.Get<TOrder>(42);
```

Is `LOrder = LOtherOrder`? With most ORMs, no — you get two distinct objects holding the same data. With Trysil and the identity map on, yes — you get the *same* object, because Trysil remembered giving you the first one and handed it back.

This is the identity map pattern. It's powerful, it prevents a class of bugs, and it introduces a different class of bugs if you don't understand what it's doing. This post is about both sides of that trade.

## What the identity map is

Conceptually, it's a dictionary: `(entity type, primary key) → object reference`, scoped to a single `TTContext`.

```
( TOrder, 42 ) → reference to the TOrder instance we returned earlier
( TOrder, 43 ) → another instance
( TCustomer, 7 ) → a TCustomer instance
...
```

Every `Get<T>` or `Select<T>` that Trysil processes consults this map first. If there's a hit, the existing instance is returned. If not, Trysil hydrates a new instance from the row, adds it to the map, and returns it.

On the write side, `Update<T>` and `Delete<T>` operate on the instance in the map — so if two parts of your code hold references to the same logical row, they see the same object, and the save is unambiguous.

## The default and how to turn it off

The identity map is **on** by default. The one-argument constructor enables it:

```pascal
LContext := TTContext.Create(LConnection);          // AUseIdentityMap := True
```

To turn it off, pass the flag explicitly:

```pascal
LContext := TTContext.Create(LConnection, False);
```

The same pattern applies to the read/write-split constructor — the flag is the last argument:

```pascal
LContext := TTContext.Create(LReadConn, LWriteConn, False);
```

When on, everything described above happens. When off, every `Get` and every `Select` hydrates fresh instances and the caller owns them.

## Why it's scoped to the context, not the process

Trysil's identity map is attached to a `TTContext` instance. Not a connection, not a process, not a thread. Each context owns its own map. When the context is freed, the map is freed with it.

This is a deliberate design decision and it matters for multi-tenant applications.

Imagine a naïve implementation where the identity map is a process-wide singleton. In a tenant-per-database model — common in SaaS — you'd have tenant A and tenant B both with rows keyed `42` in their respective `Orders` tables. A process-wide map keyed by `(TOrder, 42)` can't tell them apart. The first read fills the slot, the second read *from a different tenant* gets the wrong object, and you've just leaked cross-tenant data.

With a context-scoped map, this is impossible by construction. The tenant-A request has its own context with its own map. The tenant-B request has another. Neither can see the other's entries. No coordination needed; no risk.

The same protection applies to request scoping in any web framework: one request → one context → one map. When the request ends, everything is released.

## What it gives you

Three real benefits:

**Deduplication within a unit of work.** In a complex operation that fetches the same row through multiple paths (direct `Get`, via a `Select` filter, via a lazy-loaded relation from a parent), you get one object. Mutations are consistent.

**Cheap lazy-loading.** The lazy-loading post ended with a warning about N+1. The identity map softens that warning. A hundred orders pointing at twenty customers → the first ten unique `Customer.CompanyName` accesses hit the database; the remaining ninety are served from the map. Still not as good as `[TJoin]`, but much better than N queries.

**Clearer write semantics.** When you `Update<TOrder>(x)` and later do `LContext.Get<TOrder>(x.ID)`, you get the object you just updated, reflecting the updated state. No cache invalidation to think about.

## What it hides

The same mechanism that gives you consistency can hide bugs if you assume too much.

**Staleness across contexts.** The map is per-context. If tenant A updates `Order 42` through one context, and tenant A then creates a *new* context and reads `Order 42`, the new context fetches from the database — fine. But if your app holds long-lived contexts (or context per user session), two users editing the same row through different contexts will see their own local versions until one of them writes. This is what optimistic locking (the `[TVersionColumn]` post) is for. The identity map does not replace it.

**The list you hold is the list the context holds.** A `TTList<TOrder>` returned from `Select` is, with the identity map on, *not owned* by the list — the context owns the entities, and the list is just a container. If you free the list, the entities survive (the list doesn't `OwnsObjects`). If you free them yourself, the context's map now points at freed memory. Either respect the ownership or turn the identity map off; don't mix the two mental models.

**JOIN entities opt out.** Join entities (`[TJoin]`-based classes) are excluded from the identity map. The same logical primary key can appear in multiple result rows with *different joined-table data*, so caching by PK would collapse rows incorrectly. Join entities always come back fresh. This is described in the JOIN post; mentioned here because it's the one case where the map doesn't apply and it's easy to assume it does.

## When to turn it off

A useful rule: leave the identity map **on** (the default) for any context that lives longer than a single read — a user session, a long-running background job processing one logical batch. Turn it **off** when the context is read-only and short-lived — a one-shot export, a report generator, a data dump.

The `TTJSonContext` (next series of posts) deliberately *forbids* the identity map — it raises if you pass `AUseIdentityMap := True`. Serialization touches many entities once, doesn't benefit from deduplication, and doesn't want the extra lifetime considerations. The error message you see if you try (`'Identity map is not supported'`) is not a bug; it's a design choice.

## Testing with the identity map on vs. off

Two patterns worth knowing:

- **On.** Assert that `Get(id) == Get(id)` — same reference, not just equal state. Proves the map is doing its job. Also useful: change a field through one reference, fetch again, observe the change on the second reference.
- **Off.** Assert the opposite — two fetches produce two objects, each reflecting the database state at its fetch time. Verifies the ORM isn't accidentally deduplicating when you told it not to.

The SQLite in-memory test strategy (post at the end of the series) runs both configurations across the same test suite. That's the payoff of having the map be a flag instead of a hard-coded choice.

## Closing

The identity map is a small feature with a disproportionate effect on the feel of an ORM. With it on, your code treats entities like values in a runtime graph: the same logical row is always the same object, mutations are immediately visible, lazy relations are cheaper. With it off, every fetch is a snapshot and the caller owns what it got.

Trysil's choice — scoped to the context, on by default per context, off for JSON — threads the useful path: you get the benefits in the shapes where they apply, and nothing leaks where it shouldn't.

Next: `TTSession<T>` in practice — the unit-of-work pattern using the full-clone approach we argued for back in post #3.
