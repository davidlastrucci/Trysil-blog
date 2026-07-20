---
title: "Validation attributes and BeforeInsert/BeforeUpdate hooks"
date: 2026-07-05 08:41:00 +0200
categories: [Tutorials]
tags: [validation, attributes, hooks, tutorial]
---

Most ORMs treat validation as someone else's problem. You write your entities; you write your service layer; somewhere in that service layer there's a `Validate(entity)` method that nobody maintains because nobody knows where to put the rules.

Trysil makes a different bet: validation belongs on the entity, right next to the column definition. Two kinds of rules cover almost everything — declarative *what* rules as field attributes, and imperative *how* rules as method hooks.

## Declarative rules: one attribute per constraint

The validation attributes live in `Trysil.Validation.Attributes.pas`. Each one decorates a field, and the resolver runs them all before executing the corresponding `INSERT` or `UPDATE`.

```pascal
[TTable('Customers')]
TCustomer = class
strict private
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TRequired]
  [TMaxLength(80)]
  [TColumn('CompanyName')]
  FCompanyName: String;

  [TRequired]
  [TEMail]
  [TColumn('Email')]
  FEmail: String;

  [TRange(0, 1000000)]
  [TColumn('CreditLimit')]
  FCreditLimit: Double;

  [TRegex('^[0-9]{10,15}$', 'Phone must be 10-15 digits')]
  [TColumn('Phone')]
  FPhone: String;
end;
```

On every `Insert<TCustomer>` / `Update<TCustomer>`, Trysil walks those attributes, collects every failure into a `TTValidationErrors` list, and — if the list is non-empty — raises `ETValidationException` *before* any SQL runs. No partial writes, no half-validated state.

## The full attribute set

| Attribute | What it checks |
|---|---|
| `[TRequired]` | Not null / not empty for strings, objects, `TDateTime`, `TTNullable<T>` |
| `[TMaxLength(n)]` | String length ≤ n |
| `[TMinLength(n)]` | String length ≥ n |
| `[TMinValue(x)]` | Numeric value ≥ x (Integer or Double overloads) |
| `[TMaxValue(x)]` | Numeric value ≤ x |
| `[TLess(x)]` | Numeric value < x |
| `[TGreater(x)]` | Numeric value > x |
| `[TRange(min, max)]` | Numeric in [min, max] |
| `[TRegex(pattern)]` | String matches regex |
| `[TEMail]` | String matches the built-in email regex |
| `[TDisplayName('Label')]` | Substitutes the column name in error messages |

All the validating attributes have an overload that takes a custom error message — useful when the default (`'Field X is required'`) is too generic for a user-facing form.

```pascal
[TRequired('Company name is mandatory')]
[TMaxLength(80, 'Company name cannot exceed 80 characters')]
[TColumn('CompanyName')]
FCompanyName: String;
```

`[TDisplayName]` is the companion that keeps messages readable when columns don't match user-facing labels:

```pascal
[TDisplayName('Credit limit')]
[TRange(0, 1000000)]
[TColumn('CreditLimit')]
FCreditLimit: Double;
```

Now the error reads *"Credit limit must be between 0 and 1000000"* instead of `"CreditLimit ..."`.

## Errors, not exceptions

Here's the part that matters for UI code: `ETValidationException` carries *all* failures, not just the first one:

```pascal
try
  LContext.Insert<TCustomer>(LCustomer);
except
  on E: ETValidationException do
  begin
    for LError in E.Errors do
      Writeln(Format('%s: %s', [LError.ColumnName, LError.Message]));
  end;
end;
```

A form with five invalid fields generates five error entries in one raise. The user sees all of them at once instead of playing whack-a-mole with one error per submit.

## Validating without writing

Sometimes you want to validate before committing — say, to light up the form in red as the user types. `TTContext` exposes `Validate<T>`:

```pascal
var
  LErrors: TTValidationErrors;
begin
  LErrors := LContext.Validate<TCustomer>(LCustomer);
  try
    if LErrors.Count > 0 then
      ShowErrorsInUI(LErrors);
  finally
    LErrors.Free;
  end;
end;
```

No SQL runs. The validator only evaluates the attributes.

## When attributes aren't enough: method hooks

Some rules depend on the entity as a whole, not on a single field. "An invoice's total must equal the sum of its lines." "A customer promoted to VIP must have at least one past order." "A user's password must not match any of their last five." Attributes can't express those — they see one field at a time and have no access to the context.

For those, Trysil has method-level hooks. Declare a method, decorate it with `[TBeforeInsert]` or `[TBeforeUpdate]`, and the resolver calls it before the corresponding operation:

```pascal
[TTable('Invoices')]
TInvoice = class
strict private
  [TPrimaryKey]
  [TColumn('ID')]
  FID: TTPrimaryKey;

  [TColumn('Total')]
  FTotal: Double;

  [TColumn('LineTotal')]
  FLineTotal: Double;
public
  [TBeforeInsert]
  [TBeforeUpdate]
  procedure CheckTotals;
end;
```

```pascal
procedure TInvoice.CheckTotals;
begin
  if Abs(FTotal - FLineTotal) > 0.01 then
    raise ETException.Create(
      'Invoice total does not match the sum of the line items');
end;
```

The method is a plain instance method — it sees `Self`, it sees every field, it can do whatever Delphi code can do. If it raises, the operation is aborted the same way a validation attribute failure would abort it.

## The six hooks

| Attribute | Fires when |
|---|---|
| `[TBeforeInsert]` | Right before `Insert<T>` runs SQL |
| `[TAfterInsert]` | Right after `Insert<T>` commits |
| `[TBeforeUpdate]` | Right before `Update<T>` runs SQL |
| `[TAfterUpdate]` | Right after `Update<T>` commits |
| `[TBeforeDelete]` | Right before `Delete<T>` runs SQL |
| `[TAfterDelete]` | Right after `Delete<T>` commits |

The *Before* hooks are where validation logic belongs. The *After* hooks are where side-effects belong — writing an audit log, queuing a notification, invalidating a cache. We'll come back to *After* hooks in the dedicated events post.

## Same method, multiple attributes

A method can carry more than one hook attribute. The `CheckTotals` method above validates on both insert and update with one implementation — common enough that the attribute stacking is idiomatic:

```pascal
[TBeforeInsert]
[TBeforeUpdate]
procedure CheckTotals;
```

If the rule differed between insert and update, you'd split into two methods. But when the rule is "the totals must always match", one method with two hooks is the right shape.

## Why declare, not subclass

You could achieve the same thing with `override`s on a base entity class — a virtual `Validate` method, or a virtual `BeforeInsert`. Trysil doesn't go that way for one reason: attributes don't force inheritance.

Your `TInvoice` doesn't have to extend `TTrysilEntity` or anything else. It's a plain class. The resolver reads the attributes with RTTI and invokes whatever it finds. You can build entities that inherit from `TPersistent`, from a legacy base class, from `TObject` — it doesn't matter. The validation lives in the attributes, not in the class hierarchy.

That's the same philosophy that runs through the rest of the framework. `[TTable]`, `[TColumn]`, `[TPrimaryKey]`, `[TJoin]` — nothing needs you to inherit from a Trysil base. Attributes describe the wiring; the framework does the work.

## Closing

Validation attributes cover the declarative 90%: required, lengths, ranges, patterns, email. Method hooks cover the imperative 10%: cross-field rules, lookups, anything that reads more than one property. Both run before any SQL executes, both produce a single `ETValidationException` with all the failures at once, neither needs your entity to extend a Trysil base class.

Next: the full event lifecycle — six attributes, one factory, every side-effect your resolver can invoke.
