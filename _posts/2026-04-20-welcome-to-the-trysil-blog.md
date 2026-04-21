---
title: "Welcome to the Trysil Blog"
date: 2026-04-20 12:00:00 +0100
categories: [Announcements]
tags: [delphi, orm]
---

This is the first post of the **Trysil Blog** — the place where I'll share tutorials, deep dives,
release notes, and design decisions behind [Trysil](https://github.com/davidlastrucci/Trysil),
an ORM framework for Delphi.

## What you'll find here

- **Tutorials** — step-by-step walkthroughs, from the first entity mapping to multi-tenant REST APIs built on top of `Trysil.Http`.
- **Deep dives** — how features work under the hood: the identity map, JOIN queries, change tracking, soft delete, lazy loading, and more.
- **Release notes** — what's new, what changed, and what's on the roadmap.
- **Case studies** — real-world patterns, lessons learned, and Delphi-specific design
  choices (and why some "obvious" alternatives don't work in this language).

## A small example

```pascal
[TTable('Customers')]
TCustomer = class
strict private
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TColumn('CompanyName')]
  FCompanyName: String;

  [TVersionColumn]
  [TColumn('VersionID')]
  FVersionID: TTVersion;
public
  property ID: TTPrimaryKey read FID;
  property CompanyName: String read FCompanyName write FCompanyName;
  property VersionID: TTVersion read FVersionID;
end;
```

```pascal
var
  LCustomers: TTList<TCustomer>;
  LCustomer: TCustomer;
begin
  LCustomers := TTList<TCustomer>.Create;
  try
    FContext.SelectAll<TCustomer>(LCustomers);
    for LCustomer in LCustomers do
      Writeln(LCustomer.CompanyName);
  finally
    LCustomers.Free;
  end;
end;
```

Stay tuned — more coming soon.
