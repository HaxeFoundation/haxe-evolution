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

### Test a pattern with `is match`

```
$value is match $pattern [where $guard]
```

With `$ifBody` being `true` and `$elseBody` - `false`.

```haxe
function isSome(option: Option<Int>): Bool {
  return option is match Some(_);
}

function isDivisibleSome(option: Option<Int>, divisor: Int): Bool {
  return option is match Some(int) where (int % divisor == 0);
}
```

### Use a pattern as condition with `if match`

```
if $value match $pattern [where $guard] $ifBody [else $elseBody]
```

```haxe
function ifSome(option: Option<Int>, then: (int: Int) -> Void): Void {
  if option match Some(int) {
    then(int);
  }
}

function ifDivisibleSome(option: Option<Int>, divisor: Int, then: (int: Int) -> Void, or: () -> Void): Void {
  if option match Some(int) where (int % divisor == 0) {
    then(int);
  } else {
    or();
  }
}
```

### Capture pattern variables with `capture match`

```
$value capture match $pattern [where $guard] [else $elseBody]
```

`$ifBody` returns a structure with captured variables.
`$elseBody` throws an exception by default.

```haxe
function getSome(option: Option<Int>): Int {
  var some = option capture match Some(int);
  return some.int;
}

function getNullSome(option: Option<Int>): Null<Int> {
  var some = option capture match Some(int) else {int: null};
  return some.int;
}

function getDivisibleSome(option: Option<Int>, divisor: Int): Int {
  var some = option capture match Some(int) where (int % divisor == 0);
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
