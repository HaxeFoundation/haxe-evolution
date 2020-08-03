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

### Extract pattern variables with `extract match`

```
[$vars =] $value extract match $pattern [where $guard] [else $elseBody]
```

`$vars` acts as a whitelist for exposing pattern's captured variables.
`$elseBody` throws an exception by default. 
`$ifBody` being everything that follows in a block.

```haxe
function getSome(option: Option<Int>): Int {
  final int = option extract match Some(int);
  return int;
}

function getNullSome(option: Option<Int>): Null<Int> {
  final int = option extract match Some(int) else return null;
  return int;
}

function getDivisibleSome(option: Option<Int>, divisor: Int): Int {
  final int = option extract match Some(int) where (int % divisor == 0);
  return int;
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
