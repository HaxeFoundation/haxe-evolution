# From-to shortcut

* Proposal: [HXP-NNNN](NNNN-from-to-shortcuts.md)
* Author: [Dmitrii Maganov](https://github.com/vonagam)

## Introduction

New short syntax for allowing both from and to cast.  
New syntax for referencing underlying type in a cast.

## Motivation

Reduced repetetion and improved readability for two really common scenarios.

## Detailed design

Syntax for allowing from and to cast:

```haxe
abstract Foo(Int) from to Int {}
// equals
abstract Foo(Int) from Int to Int {}
```

Syntax for referencing underlying type:

```haxe
abstract Foo(Int) from this {}
// equals
abstract Foo(Int) from Int {}
```

## Impact on existing code

None.

## Drawbacks

Some might say that one can forget a type between `from` and `to`...

## Alternatives

Repetetion.

## Unresolved questions

Which variant to choose for from/to cast.
