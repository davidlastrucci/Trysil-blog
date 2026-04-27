---
title: "The design patterns inside Trysil"
date: 2026-04-27 08:05:00 +0200
categories: [Design Notes]
tags: [design-patterns, architecture, poeaa, gof, internals]
---

An ORM is one of those rare codebases where almost every textbook design pattern shows up because the problem *forces* it. You can't build something that maps objects to relational tables, defers loads, tracks changes, and abstracts over multiple database dialects without reaching for Data Mapper, Unit of Work, Identity Map, Strategy, Factory, and a dozen more. The patterns aren't decoration; they're answers to questions the domain keeps asking.

This post is a guided tour of those answers inside Trysil. For each pattern I'll point at the class where it lives, the problem it solves, and why *this* pattern was the right fit in Delphi specifically. Roughly: Martin Fowler's ORM patterns from *Patterns of Enterprise Application Architecture* (PoEAA) first, then the classic Gang-of-Four patterns, then a handful of adaptations that are shaped by Delphi's language constraints.

If you read the codebase with the lens "which problem forced this pattern?", a lot of things that look like arbitrary choices stop looking arbitrary.

---

## Part 1 — The ORM patterns (Fowler, PoEAA)

### 1. Data Mapper

**Where:** `TTProvider` (`Trysil.Provider.pas`) for reads and `TTResolver` (`Trysil.Resolver.pas`) for writes.

**Problem:** mapping between an in-memory object graph and relational rows is non-trivial and changes over time. You want to localise the translation so the domain objects and the database schema can evolve independently.

**Why Data Mapper, not Active Record:** Active Record would require entities to inherit a framework base class (`TTEntity` or similar) that carries the persistence methods. That conflicts directly with Trysil's attribute-based approach, where entities are plain classes decorated with `[TTable]`, `[TColumn]`, `[TPrimaryKey]`. A `TOrder` that extends nothing lets you slot Trysil into codebases that already have their own class hierarchy. The mapper sits beside the entity rather than inside it, and the split between `TTProvider` (reads) and `TTResolver` (writes) lets read-only and write-heavy code paths evolve separately.

Java calls this a POJO — Plain Old Java Object. Delphi developers less frequently use the term PODO (Plain Old Delphi Object), but the idea is identical: the entity belongs to your domain, not to the framework. A framework that forces entities to inherit from a base class like `TEntity` — as some ORMs do — breaks that property; one that works purely through RTTI attributes preserves it. Your `TOrder` stays portable: if one day you replace the persistence layer, you don't rewrite the domain.

### 2. Unit of Work

**Where:** `TTSession<T>` (`Trysil.Session.pas`).

**Problem:** a user edits five fields across three entities and clicks Save. The UI code doesn't want to reason about which entities changed — it wants to describe the new state and let the framework compute the minimum transaction that gets there.

**Why Unit of Work:** fundamentally because atomic multi-entity edits need a single transaction, and partitioning "what changed since we loaded" has to happen somewhere. Putting that logic in the session — not in every service method — keeps the UI clean. Trysil's implementation is distinctive: it clones the entire entity on entry and compares by field on commit (see *Why Trysil uses full-clone in TTSession*). That's unusual — most ORMs use change-interception proxies — but it's the only honest shape in a language without runtime proxy generation.

### 3. Identity Map

**Where:** `TTIdentityMap` (`Trysil.IdentityMap.pas`).

**Problem:** two parts of the code fetch the same row and get two different objects. Updating one doesn't update the other. Surprise.

**Why Identity Map:** to guarantee *one object per (type, primary key) per unit of work*. The key design choice — the one that matters most for real applications — is that the map is **scoped to the context**, not the process. Each `TTContext` owns its map; when the context dies, the map dies with it. In multi-tenant scenarios this is the difference between "safe by construction" and "one bug away from leaking customer A's data into customer B's response". A global singleton identity map would make that leakage inevitable; a context-scoped one makes it impossible.

### 4. Lazy Load (Virtual Proxy variant)

**Where:** `TTLazy<T>` and `TTLazyList<T>` (`Trysil.Lazy.pas`).

**Problem:** an entity has related entities. Sometimes the caller needs them, sometimes they don't. Fetching always wastes queries; fetching never is a broken abstraction.

**Why this shape, not a transparent proxy:** the textbook Virtual Proxy pattern generates a runtime subclass of the target type that intercepts every access. Java, C#, Ruby — all fine. Delphi does not have runtime subclass generation. So Trysil inverts the relationship: the parent holds an explicit `TTLazy<TCustomer>` field, and the property getter reads from it.

The elegant bit is that the framework — not user code — owns the lazy's lifetime. You declare the field with a `[TColumn('CustomerID')]` attribute on it (the lazy *is* the foreign-key column, no separate `FCustomerID` field needed), and `TTRttiMember.CreateObject` in `Trysil.Rtti.pas` instantiates the lazy during entity hydration. You never call `TTLazy<TCustomer>.Create` yourself; the getter is just `result := FCustomer.Entity` and the setter is just `FCustomer.Entity := AValue`. Lifetime follows the owning entity. The pattern is textbook Virtual Proxy with an unusual twist — the proxy is a *member* instead of a *subclass*, because Delphi makes the subclass option unavailable.

### 5. Query Object

**Where:** `TTFilter` (the query object itself) and `TTFilterBuilder<T>` (the builder that composes it), both in `Trysil.Filter.pas`.

**Problem:** queries as string concatenation are SQL-injection bait and brittle under refactoring. You want a first-class representation that can be passed around, composed, reused, serialized, and executed later.

**Why Query Object:** so the query is a value. `TTFilter` is a Delphi record — cheap to construct, cheap to pass, immutable once built (apart from appending parameters). You can build it in one method, pass it to `SelectCount` for a total, pass the same record to `Select` for the page — same query, two uses. The builder layer on top adds type-safety: `TTFilterBuilder<T>` reads the column metadata at construction time and refuses unknown column names, so a renamed column raises immediately rather than failing at query time.

### 6. Metadata Mapping

**Where:** the attribute system in `Trysil.Attributes.pas` plus the cache in `Trysil.Mapping.pas` / `TTMapper`.

**Problem:** for each entity, the framework needs to know the table name, primary key, column mappings, version column, sequence name, change-tracking roles, joins. This metadata could live in XML files, in a central config, or — the modern answer — alongside the code it describes.

**Why attributes:** because they sit next to the fields they describe, which makes the mapping visible at the point of declaration. There's no separate file to forget to update, no runtime registration dance. RTTI reads the attributes once per type (cached via `TTMapper.Instance`), and all of Trysil's machinery downstream — SQL generation, JSON serialization, HTTP routing — reads from the same cached maps. One source of truth, one read.

### 7. Optimistic Offline Lock

**Where:** `[TVersionColumn]` + `TTVersion` (type) + the SQL generated by `TTGenericUpdateCommand` and friends.

**Problem:** two users load the same row at the same time, both edit, both save. Without protection, the second save overwrites the first silently.

**Why optimistic, not pessimistic:** pessimistic locking (holding a row lock across the UI edit) doesn't scale and doesn't survive disconnected clients. Optimistic locking trades nothing by default: a small integer per row, one extra predicate in the `WHERE`, and a conflict is detected at commit time. Trysil's version column is always application-managed (the resolver increments it); this keeps the mechanism portable across SQLite / PostgreSQL / Firebird / SQL Server without depending on any single DB's `ROWVERSION` / trigger infrastructure.

---

## Part 2 — GoF classics

### 8. Facade

**Where:** `TTContext` (`Trysil.Context.pas`).

**Problem:** behind a working ORM sit many collaborators — a provider, a resolver, a mapper, an identity map, connection(s), a transaction manager, a factory for sessions. If the consumer had to instantiate and wire them all, the surface area would be overwhelming.

**Why Facade:** so users interact with one object. `TTContext.Create(connection)` is the whole setup; everything else is method calls (`Get<T>`, `Select<T>`, `Insert<T>`, `CreateSession<T>`, …). The complexity behind the facade is internal — rearranging it doesn't break the consumer. This is what "Trysil is easy to learn" reduces to at the architectural level: the facade is designed to stay small even as the internal machinery grows.

### 9. Strategy

**Where:** `TTSyntaxClasses` in `Trysil/Data/SqlSyntax/`, one implementation per database.

**Problem:** `SELECT`-with-paging is different in PostgreSQL (`LIMIT/OFFSET`), SQL Server (`OFFSET ... FETCH NEXT` with mandatory `ORDER BY`), and Firebird (`FIRST/SKIP`). Identity retrieval is different everywhere. Sequence syntax is different everywhere. And SQLite doesn't have real sequences at all.

**Why Strategy:** the variable parts of SQL generation are pulled into a strategy object that the connection holds. Each driver (`TTSQLiteConnection`, `TTPostgreSQLConnection`, …) instantiates the syntax classes it wants; the rest of the framework doesn't care which. Switching databases is swapping the strategy; no conditional blocks in the generic resolver or provider code.

### 10. Bridge

**Where:** `TTConnection` (the abstraction) vs. FireDAC drivers (the implementor).

**Problem:** two orthogonal hierarchies need to vary independently — *what SQL shape Trysil emits* on one axis, *what wire protocol is spoken* on the other. If they weren't decoupled, adding a new SQL dialect would force touching every wire-level driver, or vice versa.

**Why Bridge, not Adapter:** Adapter would make sense if we were wrapping a fixed external interface to fit our own. Bridge is the right pattern here because *both* hierarchies are ours to evolve — the connection abstraction can gain methods, and FireDAC drivers can be swapped, without either dragging the other. The bridge is literally the composition: `TTFireDACConnection` holds a FireDAC `TFDConnection` and delegates protocol-level work to it while owning SQL-level work itself.

### 11. Singleton

**Where:** `TTMapper.Instance`, `TTFactory.Instance`, `TTEventFactory.Instance`, `TTLogger.Instance`, `TTLanguage.Instance`.

**Problem:** things that are genuinely global — the cache of entity mappings, the factory that creates event objects, the I18N resource strings — shouldn't be re-derived per context. Re-reading RTTI for every `Get<TOrder>` would be wasteful.

**Why Singleton (and not per-context caches):** the mapping from `TOrder` to its table metadata is the same everywhere. It's immutable data once computed. The trade-off is thread safety — the singletons that are written to (the event factory's method cache, the mapper's table cache) use a `TTMultiReadExclusiveWriteLock` so readers don't block each other and writers happen rarely. Delphi's `class constructor` / `class destructor` provide clean construction and teardown for these — no explicit `Initialize` / `Finalize` calls anywhere.

### 12. Factory Method

**Where:** `TTContext.CreateEntity<T>`, `CreateEntityList<T>`, `CreateSession<T>`, `CreateTransaction`, `CreateFilterBuilder<T>`, `CreateDataset`; plus `TTEventFactory.CreateEvent<T>`.

**Problem:** instantiation depends on context state — which connection, whether the identity map is on, which metadata to load, which event constructor to invoke. Exposing `TTSession<T>.Create(LContext, LTableMap, …)` would force consumers to know all the wiring. That's exactly the coupling a factory method removes.

**Why Factory Method:** because the context owns the knowledge of its state, it's the only thing that knows enough to instantiate the collaborator correctly. `CreateSession<T>` can pass the right metadata, the right identity-map flag, the right connection. `TTEventFactory.CreateEvent<T>` uses RTTI to find a constructor matching `(TObject, T)` — which is how Trysil dynamically wires user-defined event classes. That dispatch couldn't live on the consumer side.

### 13. Builder

**Where:** `TTFilterBuilder<T>` (`Trysil.Filter.pas`).

**Problem:** a `TTFilter` has multiple optional parts — a WHERE clause that's a composition of conditions, parameters, ordering, limit/offset, the include-deleted flag. Assembling these by hand means multiple constructor overloads and error-prone manual sequencing.

**Why Builder:** the fluent API (`.Where(...).Equal(...).AndWhere(...).OrderByAsc(...).Limit(...).Build`) lets you describe exactly what you want step by step, with each step returning the builder for the next. The builder keeps internal state that turns into the final immutable record at `Build`. Also, the builder is where entity-aware validation lives — unknown column names raise `ETException` at the `.Where('Typo')` call site, not at query time.

### 14. Template Method

**Where:** `TTEvent` (`Trysil.Events.Abstract.pas`) with `DoBefore` / `DoAfter`; `TValidationAttribute` with its `Validate` and the abstract `IsValid` / `AddValidationError` in specialisations like `TLengthAttribute`, `TValueAttribute`.

**Problem:** a family of related algorithms share the same skeleton but differ in one or two steps. Copying the skeleton per subclass is duplication; making it a free function passes in too many parameters.

**Why Template Method:** the base class owns the algorithm (for validation: "if the value fails type check, add an error; otherwise delegate to `IsValid` and if false, add a domain-specific error"). Subclasses only implement the *varying* pieces. `TMaxLengthAttribute` provides its own `IsValid` (`AValue.Length <= FLength`) and its own `AddValidationError` message. The skeleton is written once. This is what keeps the validation attribute set consistent — every attribute follows the same `Validate` contract because the base enforces it.

### 15. Command

**Where:** `TTDataAbstractCommand` and its subclasses (insert / update / delete / soft delete), created by the connection.

**Problem:** a write operation is conceptually *"do this operation against this entity on this connection"*. Passing three things through multiple layers is awkward. A single command object that knows how to execute itself is cleaner.

**Why Command:** the resolver doesn't need to know how to generate `INSERT` syntax or how to extract the just-created identity — it asks the connection for a command and calls `.Execute` on it. The command subclass knows its own shape. Also, this is where soft-delete abstraction lives: when the entity has `[TDeletedAt]`, `CreateSoftDeleteCommand` returns an object that emits `UPDATE`, not `DELETE`, with the same `.Execute` contract. The resolver doesn't branch; the command does.

### 16. Observer / Event

**Where:** `TTEvent` subclasses wired via `[TInsertEvent]` / `[TUpdateEvent]` / `[TDeleteEvent]` class attributes, plus method hooks (`[TBeforeInsert]`, etc.) dispatched by the resolver.

**Problem:** the framework needs to give user code ways to react to persistence operations — before/after insert, update, delete — without forcing every entity to override virtual methods on some base class.

**Why Observer (in two shapes):** the *event class* shape is classical Observer, with the factory resolving and instantiating a subscriber per write. The *method hook* shape is a lightweight declaration — the method lives on the entity, its attributes tell the resolver to invoke it at the right moment. Both are attribute-registered, neither requires inheritance, both are dispatched by the same resolver pipeline. Giving consumers two shapes isn't mess — it's acknowledging that cross-cutting concerns (a class somewhere else) and entity-local concerns (a rule that belongs next to the fields) are different shapes of the same pattern.

---

## Part 3 — Patterns shaped by Delphi

Some of Trysil's most interesting choices aren't a clean application of a named pattern; they're adaptations the language forces. Worth calling out explicitly.

### 17. Attributes as a substitute for annotations + AOP

Java uses annotations (at declaration) plus bytecode weaving (at compile time) or dynamic proxies (at runtime) to turn declarations into behavior. A field annotated `@Column` doesn't just say "this is a column" — a proxy class generated elsewhere reads the annotation and wires the read/write path.

Delphi has attributes, readable via RTTI, but no weaving and no proxies. Trysil adapts: **the framework reads the attributes from its own code and does directly what a proxy would do elsewhere.** The resolver reads `[TColumn]`, the session diff reads `[TColumn]`, the JSON serializer reads `[TColumn]`. No proxy, no weaving, no surprise code generation — the framework explicitly consults the metadata at every use site. More code in the framework, zero code generation at compile or run time.

### 18. RAII-like lifecycle with auto-commit/rollback

`TTTransaction` (`Trysil.Transaction.pas`) auto-starts on `AfterConstruction` and auto-commits on `BeforeDestruction`. If you want a rollback instead, you call `Rollback` explicitly before the object dies.

**Why:** Delphi's `try/finally/Free` is the language's RAII equivalent. Wiring the transaction lifecycle to object lifetime means you can't forget to commit (it happens on scope exit), and you can't accidentally leave a transaction open (the `Free` commits). The one escape — `Rollback` — is explicit by design: not committing should be the exception, not the default.

### 19. Full-clone diff instead of field interception

Repeating from the full-clone post for completeness. Change tracking in most ORMs is implemented with field-level interceptors — runtime proxies, or bytecode rewrites, that flag a field as dirty on write. Delphi has neither.

Trysil's `TTSession<T>` solves the same problem by **cloning the entire entity when it enters the session**. At `ApplyChanges`, it compares each entity to its own clone, field by field, and emits the write. More memory (two entities instead of one), but no runtime code generation, no pre-compile step, no surprise behavior on field assignment.

The alternative path — instrumented setters on a mandatory base class, as some ORMs take — would have worked at the cost of giving up PODO purity: the domain would own the framework instead of the other way around. Snapshot is more expensive in RAM; it's more honest in design.

This is the pattern of "when the textbook answer requires language features you don't have, the substitute isn't a worse implementation of the same pattern — it's a different pattern that produces the same guarantees".

### 20. Pluggable logging

`TTLogger.Instance` is a singleton that accepts writer callbacks. Events flow out as structured items (`TTLoggerItem`); where they go is the writer's call.

**Why this shape, not a fixed log target:** logging in production has to coexist with whatever aggregator the project already has — Serilog-style sinks, ELK, Datadog, file-per-day, stdout-only. Hardcoding any of these would force consumers to either accept the framework's choice or route around it. The pluggable writer pattern keeps the framework honest: emit structured events, let someone else render.

---

## The lens

Each pattern in Trysil has a specific problem behind it, and the shape of the solution usually tells you something about the constraint that forced it.

- Multi-tenant safety forced **context-scoped Identity Map**, not a global one.
- Four databases with four SQL dialects forced **Strategy** for syntax, **Bridge** between Trysil's SQL abstraction and FireDAC's wire protocol.
- Delphi's inability to host generic methods on interfaces forced **concrete `TTContext`**, which in turn forced **integration testing over SQLite in-memory** as the practical test strategy.
- Delphi's lack of runtime proxies forced **full-clone diff** in `TTSession<T>` and **explicit lazy containers** in `TTLazy<T>`.
- Attributes without weaving or proxies forced **the framework to read its own metadata** everywhere it needs it.
- Cross-cutting concerns like audit logs and message-bus emission forced the **Observer pattern in two shapes**: entity-local hooks for rules that belong near the data, event classes for things that shouldn't pollute the entity.

Patterns aren't a checklist to apply. They're a vocabulary for the shape of an answer. An ORM, by the nature of what it does, asks most of the questions in that vocabulary — which is why the codebase of any serious ORM, Trysil included, reads like a bestiary of them.

The next time you look at a class in `Trysil/` and wonder "why is it structured like this", try this: what would happen if it were structured differently? What problem shows up? The class is the answer; the problem is the question. If you can reconstruct the question from the code, you've learned both the framework and, usually, one pattern you didn't fully understand before.

That's the real value of reading a mid-sized framework's source in a language you know. Trysil is not huge. You can read the whole thing. When you do, the patterns stop being Wikipedia entries and start being moves you've seen before.

---

*This is the special post for April 27, 2026. Thanks for reading — if you have questions, corrections, or patterns I missed, [drop me a line](mailto:david.lastrucci@gmail.com) or [open an issue](https://github.com/davidlastrucci/Trysil/issues).*
