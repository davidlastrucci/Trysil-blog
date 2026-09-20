---
title: "Trysil 2.0.0: what breaks, and why we counted every one"
date: 2026-09-20 08:00:00 +0200
categories: [Announcements]
tags: [release, breaking-changes, migration, changelog]
---

Trysil 2.0.0 is out. The changelog is long, and it opens with a sentence that is not the usual release-note tone:

> This section carries **twenty-two compilation breaks** and **eighty-four breaks the compiler does not catch**.

That second number is the one worth talking about. Anybody can list the changes that stop your build - the compiler lists them for you, whether or not the release notes do. The interesting question is what a release does with the other kind: the changes that leave your code compiling perfectly and make it behave differently.

We counted those too, and wrote down all eighty-four. This post is about why, and what they are.

## The ones the compiler catches

These are the cheap ones. You upgrade, you build, you get an error with a file and a line, you fix it. Twenty-two of them, each with a paragraph in the changelog and a migration in one line. A sample:

**`TTProvider.Get<T>` lost its one-argument overload.** It is now `Get<T>(const AID, const AIncludeDeleted)`, because a `Get` on a soft-deleted row had no way to say which of the two answers it wanted.

**`TTTableMap.DetailColums` is now `DetailColumns`.** A typo in a public read-only property. Renaming it breaks code; leaving it means the typo is API forever.

**`SelectCount` returns `Int64`.** It read the count into an `Integer` from a field the engine hands back 64-bit, so a table with more than two billion rows reported a negative number of them. Only a driver that overrides the method has to change; the calls do not.

**A driver configures the connection through `DoConfigureConnection`.** `ConfigureConnection` is no longer virtual. If you derive from a Trysil connection to add an option of your own, rename your override and drop the `inherited`. That last one is not cosmetic, and we will come back to it.

You will find these. They cost you an afternoon and no thought.

## The ones it doesn't

Here is one, in full, because it is the shape of the whole category.

Trysil derives a JSON key from a Delphi member name, and the convention is that a field starts with `F`. So `FCustomerName` becomes `customerName`. The old implementation stripped the first letter whenever it was an `f` or an `F`, without looking at what followed:

```pascal
//              1.0.0            2.0.0
FCustomerName   "customerName"   "customerName"   // a real prefix: unchanged
Fax             "ax"             "fax"
FirstName       "irstName"       "firstName"
FatturaID       "atturaID"       "fatturaID"
```

The round trip held, because the same function runs in both directions: Trysil wrote `atturaID` and read `atturaID` back. Nothing failed. The only casualty was any client that was not Trysil - a TypeScript type generated from the payload, a mobile app, a partner's integration - all built on a key with a letter missing.

2.0.0 strips the prefix only when the next character is uppercase, which is what the convention actually says. A field really named `FFax` still comes out as `fax`; a member whose name simply happens to begin with an `F` now keeps it.

Now read that again as an upgrade. **Every member whose name starts with `F` or `f` and does not continue with an ASCII capital changes its key, in both directions** - a lower-case letter, a digit, an underscore. A client built on the old spelling reads empty and writes nothing. No exception, no log line, no compiler error. If your entities have a `FatturaID` or a `FirstName`, you need to know this before you deploy, and the only way you can know it is if somebody counted it as a breaking change and wrote it down.

Two more, one of them from outside the count:

**On Oracle, a `NUMBER(1)` is now read as a boolean.** Oracle has no `BOOLEAN` before 23ai, so a `Boolean` member has to live in a `NUMBER(1)`, and the driver declares that mapping to FireDAC. The rule is keyed on precision, not on the member name: *every* `NUMBER(1)` on an Oracle connection comes back as a boolean now. A status code or a counter in a column that narrow has to be widened to `NUMBER(2)`. The changelog marks this one **Breaking** and declares it apart from the count, but it asks the same of you.

**A custom driver must implement three new methods.** If you wrote a `TTGenericConnection` outside the repository - the base class is documented for exactly that - the transaction methods moved. Your driver still compiles. An unimplemented abstract is a hint where the driver is constructed and an `EAbstractError` at the first transaction, in production. Worse and quieter: a driver that overrode `CommitTransaction` keeps compiling and keeps being called, never reaches the new method, and therefore never notifies the observer that keeps the new-entity cache honest.

That last one is why `ConfigureConnection` stopped being virtual in this release rather than staying virtual with a warning in the documentation. The documentation already said "call `inherited`". A sentence in the docs is not a property of the code. Forgetting it did not produce an error, it produced **a different value**: an amount silently rounded to two decimals, or a `NUMBER(1)` read back as a number. Now it does not compile, and that is an improvement.

## Why counting matters

A breaking change you declare is a decision. A breaking change you don't is a trap, and the person who falls into it is the one who trusted you enough to upgrade.

The counting is also a discipline, not a formality. While preparing this release we went through the changelog against the code, entry by entry, and found four places where the changelog claimed a guarantee the code did not give. One of them said identifiers were quoted everywhere; they were quoted inside the SQL generator and not in the filter builder or the lazy loader. That is worse than saying nothing, because a host reads the line and stops defending themselves. All four are now either true or reworded.

The count itself was wrong, too. It said nine compilation breaks and seven silent ones. Every pass over the section moved both numbers: a break that had been described inside another entry instead of getting its own, silent changes the entries described without ever counting, and a last compilation break added by a fix made while these notes were being written. Then two rounds of external audit and our own review agents moved it again, and every move had a reason: three public members that nothing called were removed instead of documented, an exact decimal read through the dataset path keeps its digits instead of the fifteen a `Double` is written with, and a date read the same way stops arriving as the evening of the day before. It says twenty-two and eighty-four now. The largest move came from applying the rule for what counts to every entry rather than to the ones we happened to look at, one at a time, against the tag and against the public master: it found dozens of silent breaks the section had described in full and never counted. The later rounds moved it in both directions. An entry with no **Breaking** label is where a break hides best - an `ORDER BY` that used to take any text and now takes a mapped column, an overload with no `[TArea]` that used to inherit its sibling's areas and now answers every authenticated caller - and four of those came in that way. Two went out again when it turned out the only way to reach them was an overload this release introduces, which no existing code can be calling. That is the number the method produces, and the method is the point: every entry read against the tag and against the public master, one at a time, until a pass found nothing new. Most of them are defects corrected rather than decisions taken, which is not a reason to leave them out: the question is whether your code has to change, not whether what it was written against was worth keeping. If that sounds like a small thing to fuss over, consider that the number is the first thing a reader uses to decide how carefully to read the rest.

## What makes any of this checkable

Reading your own code and agreeing with yourself is not evidence. The suite is: **more than 2300 tests, run against SQLite, PostgreSQL, Firebird, InterBase, SQL Server, MariaDB and Oracle**, on Delphi 13. The packages build on Delphi 11 and 13; the sets for 10.3, 10.4 and 12 are kept aligned but were not compiled for this release, and the changelog says so. Most of them are written once, in an abstract fixture, and executed seven times.

That is not a badge. It is the thing that keeps finding what reading does not. An example from three days before this release: an audit noted that Trysil had no test anywhere declaring a text LOB - a `CLOB`, a `TEXT`, an `nvarchar(max)` - on any engine. So we added one column to the test model and ran the seven.

Sixteen tests went red, all of them on Oracle, all with the same message: *parameter not registered for type `ftOraClob`*. Oracle reports a `CLOB` as its own field type, and that type had never been registered among the string parameters, while its binary twin two lines below had. The consequence was not "text LOBs are slow on Oracle" or "they truncate". It was that **an entity mapping a `CLOB` could not be written on Oracle at all**, and had never been able to.

Nobody had ever hit it because nobody had ever asked. The fix is one line. Finding it took one column in a test model and seven engines, and no amount of careful reading had turned it up in either of two audits.

## What you get for the afternoon

The breaks are the price. The release is the rest of it:

- **Audit columns that mean something.** `Update<T>` no longer rewrites `[TCreatedAt]` and `[TCreatedBy]`, and change tracking columns are no longer readable from a JSON body. "Created by" now answers the question it is named after.
- **`Currency` end to end.** A money value is read and written by Trysil as a fixed-point decimal on all seven engines. One `Double` is left, and it is FireDAC's own: `TCurrencyField` keeps its value in one, so a narrow `NUMERIC` that the driver hands over as that field - on PostgreSQL, for one - loses its last decimals past about 9 * 10^11, and the changelog says where. Two separate faults used to truncate the same amount, and one of them was the driver sending it as a type that keeps two decimals.
- **Identifiers are quoted in the statements Trysil builds.** Table, column, join alias, output alias, sequence, and from this release the filter builder, the `ORDER BY` and the foreign key a lazy list writes. A column called `Value`, `Level` or `Order` stops being something you have to know about. One path is left, and the changelog names it rather than claiming "everywhere": the `WHERE` text of the expression API is still written by the expression itself, unquoted.
- **Startup that refuses what cannot work.** Mappings that cannot mean anything are rejected when the entity is mapped, not when the query fails: a column that is both the primary key and the version column (its update would read `SET ID = ID + 1 WHERE ID = :ID`, moving the key it identifies by), a key column that also carries a change tracking attribute, a member carrying both `[TColumn]` and `[TDetailColumn]`, a `[TDeletedBy]` with no `[TDeletedAt]`, and an entity that maps no column at all.
- **Text LOBs on all seven.** A `CLOB`, a `TEXT`, a `LONGTEXT`, an `nvarchar(max)` or a `BLOB SUB_TYPE TEXT` maps to a plain `String` member and is bound as a LOB, so a value larger than what an ordinary string bind accepts travels whole.

## Upgrading

Read the `## 2.0.0` section of [the changelog](https://github.com/davidlastrucci/Trysil) end to end. It is long on purpose, and the entries marked **Breaking** are not the whole story - that is the point of this post. The documentation has a [checklist](https://davidlastrucci.github.io/Trysil/guide/upgrading/) that lists the twenty-two and the eighty-four in the order you meet them.

Then do three things:

1. Build. Fix the twenty-two.
2. Search your entities for a member whose name starts with `F` or `f` and does not continue with `A` to `Z`. Those are your JSON keys changing.
3. If you have a driver of your own, or a `TTPostgreSQLConnection` descendant, read the four entries about custom drivers before you deploy rather than after.

If something in the list is wrong, or breaks in a way we did not describe, [open an issue](https://github.com/davidlastrucci/Trysil/issues). A break we got wrong in the notes is worth more to us than a bug report.

Next: `TTSession<T>` in practice - the unit-of-work pattern using the full-clone approach we argued for back in post #3.
