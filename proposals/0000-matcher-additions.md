# Matcher additions: pattern.set and pattern.where

* Proposal: [HXP-NNNN](NNNN-matcher-additions.md)
* Author: [Dmitrii Maganov](https://github.com/vonagam)

## Introduction

Two new features for matcher: way to set variables in a pattern without capturing (`pattern.set`) and ability to add a guard by a expression in a pattern (`pattern.where`).

## Motivation

Setting variables is for or-patterns where you want to reuse case body for similar patterns but one of them lacks some variable that other captures or if the variable in question represents something that cannot be gained by means of extraction.

Guard as a expression helps or-patterns too since it makes possible to nest it, but also allows to express full case condition as a single haxe expression, which in turn potentially simplifies design needed for a single pattern check.

## Detailed design

Examples for setting variables:

```haxe
switch (option) {
  case 
    Some(int) | 
    None.set(int = 0)
  :
    // common code here
}
```

```haxe
switch [optionA, optionB] {
  case [Some(_), Some(_)] | [None, None]:
  case
    [Some(int), None].set(onLeft = true) |
    [None, Some(int)].set(onLeft = false)
  :
    // common code here
}
```

```haxe
switch (expression) {
  case
    (macro final $name = ${value}).set(isFinal = true) |
    (macro var $name = ${value}).set(isFinal = false) |
    (macro $i{name} = ${value}).set(isFinal = false)
  :
    // common code here
  case _:
}
```

Example for guard expression:

```haxe
switch (either) {
  case 
    Left(int).where(int < 0) | 
    Right(int).where(int > 0)
  :
    // common code here
  case _:
}
```

Initially went for operators following example of `|`, but a method call is easier to read and understand, no need to worry about precedence.

## Impact on existing code

None.

## Drawbacks

Right now switch expression produces duplication of code in output for or-patterns, and as the proposal makes it easier to use them it might unknowingly encourage less pretty output.

But this is the problem with or-patterns (and potentially solvable), not with the proposal itself.

## Alternatives

Without this the only choice is to move common case code to a function and just call this function in a switch.

## Opening possibilities

Guard as an expression and not a separate entity helps with single pattern shortcut desing removing third wheel from current value-pattern-guard combo. Both `pattern.set` and `pattern.or` aid or-patterns, because of that more things can be represented in a single pattern, which improves usability of the potential shortcut.

## Unresolved questions

None?
