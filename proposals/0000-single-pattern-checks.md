# Single pattern checks

* Proposal: [HXP-NNNN](NNNN-single-pattern-checks.md)
* Author: [Dmitrii Maganov](https://github.com/vonagam)

## Introduction

New syntax for single pattern checks.

## Motivation

Current single pattern check - `match` method on an enum value - is limited in functionality compared to usual pattern matching. No captures or guards and only a single enum value as a root value.

Proposed syntax will allow to use full arsenal of the matcher.

## Detailed design

### Test a pattern

Get a boolean result with:

```haxe
$value is case $pattern

// same as

switch ($value) {case $pattern: true; case _: false;}
```

Usage:

```haxe
function isSome(option: Option<Int>): Bool {
  return option is case Some(_);
}

function isDivisibleSome(option: Option<Int>, divisor: Int): Bool {
  return option is case Some(int).where(int % divisor == 0);
}
```

### Use captured variables in a conditional body

If an `if` or `while` condition is a single `is case` expression, then it is transformed like this:

```haxe
if ($value is case $pattern) {
  // access pattern variables
}

// same as

switch ($value) {
  case $pattern: {
    // access pattern variables
  }
  case _:
}
```

```haxe
while ($value is case $pattern) {
  // access pattern variables
}

// same as

while (true) {
  switch ($value) {
    case $pattern: {
      // access pattern variables
    }
    case _: break;
  }
}
```

Usage:

```haxe
function ifSome(option: Option<Int>, then: (int: Int) -> Void): Void {
  if (option is case Some(int)) {
    then(int);
  }
}

function ifDivisibleSome(option: Option<Int>, divisor: Int, then: (int: Int) -> Void): Void {
  if (option is case Some(int).where(int % divisor == 0)) {
    then(int);
  }
}
```

## Impact on existing code

None.

## Drawbacks

None?

## Alternatives

Limited `EnumValue.match` or custom macro.

## Unresolved questions

None?
