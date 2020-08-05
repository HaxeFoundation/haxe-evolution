# Single pattern checks

* Proposal: [HXP-NNNN](NNNN-single-pattern-checks.md)
* Author: [Dmitrii Maganov](https://github.com/vonagam)

## Introduction

New syntax for single pattern checks.

## Motivation

Current single pattern check - `match` method on an enum value - is limited in functionality compared to usual pattern matching. No captures or guards and only a single enum value as a root value.

Proposed syntax will allow to use full arsenal of matcher for three common scenarios.

## Detailed design

All of single pattern checks allias such switch:

```haxe
switch ($value) {
  case $pattern if ($guard): $ifBody;
  case _: $elseBody;
}
```

### Test a pattern with `case is`

```
$value case is $pattern [if ($guard)]
```

With `$ifBody` being `true` and `$elseBody` - `false`.

```haxe
function isSome(option: Option<Int>): Bool {
  return option case is Some(_);
}

function isDivisibleSome(option: Option<Int>, divisor: Int): Bool {
  return option case is Some(int) if (int % divisor == 0);
}
```

### Use a pattern as condition with `case`

```
$value case $pattern [if ($guard)]: $ifBody [else $elseBody]
```

```haxe
function ifSome(option: Option<Int>, then: (int: Int) -> Void): Void {
  option case Some(int): {
    then(int);
  }
}

function ifDivisibleSome(option: Option<Int>, divisor: Int, then: (int: Int) -> Void, or: () -> Void): Void {
  option case Some(int) if (int % divisor == 0): {
    then(int);
  } else {
    or();
  }
}
```

### Get pattern variables with `case vars`

```
$value case vars $pattern [if ($guard)] [else $elseBody]
```

`$ifBody` returns a structure with captured variables. 
`$elseBody` throws an exception by default.

```haxe
function getSome(option: Option<Int>): Int {
  var some = option case vars Some(int);
  return some.int;
}

function getNullSome(option: Option<Int>): Null<Int> {
  var some = option case vars Some(int) else {int: null};
  return some.int;
}

function getDivisibleSome(option: Option<Int>, divisor: Int): Int {
  var some = option case vars Some(int) if (int % divisor == 0);
  return some.int;
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
