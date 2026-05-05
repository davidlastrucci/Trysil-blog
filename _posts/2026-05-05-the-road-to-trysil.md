---
title: "The Road to Trysil"
date: 2026-05-05 21:51:00 +0200
categories: [Design Notes]
tags: [delphi, orm, backstory]
---

I'd wanted to write an ORM since the .NET 2.0 days — though back then I didn't even know what the word meant.

The platform had everything you needed: attributes, generics, reflection. The pieces fit together — you could picture, in your head, a tool that read your classes, mapped them to tables, and gave you a typed API to talk to a database. I never wrote it.

Back then every line of code that talked to a database was written by hand: mapping classes, Data Access Layers full of repetitive SELECTs and INSERTs, type conversions you'd write over again for every table. The kind of boilerplate that makes you think *there has to be a better way*.

One day I discovered **NHibernate** — the .NET port of Hibernate, the Java ORM that had set the bar everyone else was measuring themselves against. It worked. It was powerful. And it taught me, the way only using a thing teaches you, what an ORM actually does for you and where it pushes back when you ask too much of it.

Around the same time I read the books. Eric Evans' *Domain-Driven Design*. Martin Fowler's *Patterns of Enterprise Application Architecture*. The Unit of Work, the Identity Map, the Repository — they weren't tricks Hibernate had invented. They were patterns the field had named, and once you saw them you couldn't unsee them. They were everywhere I looked, including in code I'd already written without knowing what to call it.

Years passed. I kept writing software. I kept not writing the ORM.

What I had been writing, the whole time, was Delphi. Pascal had been my first love and Delphi never stopped being the language I reached for when I wanted to build something *real* — desktop apps, server code, full systems. And in the meantime, Delphi had quietly grown up. Attributes arrived. Generics arrived. Extended RTTI arrived — the same kind of reflection I'd taken for granted in Java and .NET.

One day I looked at the language and realised: everything I'd needed in 2005 to write the ORM was now sitting in front of me, in my favourite language.

So I started.

That's it, really. There's no dramatic origin story. I had an itch from twenty years ago and a language that finally had the parts to scratch it. Trysil is what came out of that.
