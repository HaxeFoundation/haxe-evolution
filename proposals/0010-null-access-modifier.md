# "null" access modifier vs "this"

* Proposal: [HXP-NNNN](NNNN-null-access-modifier.md)
* Author: [Dmitry Hryppa](https://github.com/haxedev)

## Introduction

Replace `null` access identifier with `this`.

## Motivation

It's a bit confusing to use `null` as a "protected" access modifier.
`this` is much understandable and gives as a context about who can modify the current field.

## Detailed design

It's only about replacing a `null` keyword with `this` as a "protected" access modifier.

Example:
```haxe
public var isDataExists(default, this):Bool;
```

## Impact on existing code

It will conflict with the old codebase. But `null` may be marked as deprecated for some time before it will be fully removed from access modifiers syntax.

## Drawbacks

.

## Alternatives

.

## Opening possibilities

.

## Unresolved questions

.
