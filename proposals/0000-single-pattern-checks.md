# Single pattern checks

* Proposal: [HXP-NNNN](NNNN-single-pattern-checks.md)
* Author: [Dmitrii Maganov](https://github.com/vonagam)

## Introduction

New syntax for single pattern checks.

## Motivation

Current single pattern check - `match` method on an enum value - is limited in functionality compared to usual pattern matching. No captures or guards and only a single enum value as a root value.

Proposed syntax will allow to use full arsenal of matcher for two common scenarios.

## Detailed design

All of single pattern checks allias such switch:

```haxe
switch ($value) {
  case $pattern if ($guard): $ifBody;
  case _: $elseBody;
}
```

### Test a pattern

```
is case $value => $pattern [if ($guard)]
```

With `$ifBody` being `true` and `$elseBody` - `false`.

```haxe
function isSome(option: Option<Int>): Bool {
  return is case option => Some(_);
}

function isDivisibleSome(option: Option<Int>, divisor: Int): Bool {
  return is case option => Some(int) if (int % divisor == 0);
}
```

### Use a pattern as condition

```
in case $value => $pattern [if ($guard)]: $ifBody [else $elseBody]
```

```haxe
function ifSome(option: Option<Int>, then: (int: Int) -> Void): Void {
  in case option => Some(int): {
    then(int);
  }
}

function ifDivisibleSome(option: Option<Int>, divisor: Int, then: (int: Int) -> Void, or: () -> Void): Void {
  in case option => Some(int) if (int % divisor == 0): {
    then(int);
  } else {
    or();
  }
}
```

## Impact on existing code

None.

## Drawbacks

None?

## Alternatives

`EnumValue.match` or custom macro.

## Unresolved questions

None?
