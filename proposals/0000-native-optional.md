# Notation for native optional arguments

* Proposal: [HXP-NNNN](NNNN-native-optional.md)
* Author: [Dmitrii Maganov](https://github.com/vonagam)

## Introduction

Syntax and proper way to specify native optional arguments.

## Motivation

Current lack of clear distinction between haxe generated optional (nullable) arguments and native (non-nullable) optional ones creates bunch of problems for working with external code and even internally for null-safety.

```haxe
?Int->Void // Can you pass a null to such signature or not?

function(value=0): Void; // What about that one?
```

## Detailed design

Proposed notation for native optional non-nullable arguments that should always be at the end of an argument list, cannot be skipped with nulls in bindings (or with exotic haxe's type skipping), cannot be null-padded, does not require providing a default value that may go out of sync with described extern, does work clearly with old function notation:

```haxe
?!Int->Void

function(?!value: Int): Void;
```

Ideally for such arguments targets should generate appropriate native not-nullable optional arguments.

## Impact on existing code

Previous convention to specify native optional arguments `(value: Int = 0)` should imply `?!value`, but if it is located before non-optional or haxe-optional should be treated as `?value` and produce the warning asking to either mark the argument with `?` or to move it.
