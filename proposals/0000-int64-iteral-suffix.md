# haxe.Int64 Numeric Literal Suffix

* Proposal: [HXP-0000](0000-int64-iteral-suffix.md)
* Author: [Aidan Lee](https://github.com/aidan63)

## Introduction

By appending an `i64` to the end of an integer literal (e.g. `final myInt64 = 1000i64;`) it will become a `haxe.Int64` instead of a standard `Int`.

## Motivation

Creating `haxe.Int64` objects is not as easy as standard integers. You either have to use `haxe.Int64.make` and manually split the number into low and high bits if your initial value is too large for a 32bit integer, or you can use the `@:from` function for getting one from an existing `Int`.

## Detailed design

The compiler could detect if a `Const(Int(s))` ends with an `i64` suffix and if so create the necessary high and low bits from `s` and insert a `haxe.Int64.make` call in place. This doesn't require any changes to the AST, only that the lexer allows `i64` suffixes for constant integers. To simplify things the `i` in the suffix is case sensitive, that is `1000I64` is not valid.

I've put together a quick proof of concept in this branch (Uses previously proposed `L` suffix). https://github.com/aidan63/haxe/tree/int64-suffix

## Impact on existing code

Macro functions which attempts to parse `EConst(CInt(s))` strings might start to fail if they encounter suffixes as this is a new concept to haxe.

## Drawbacks

There are several oddities when using `haxe.Int64` (such as keys for `haxe.ds.Map`, analyser-optimise not simplifying maths) which hasn't effected too many users so far (assumably because `haxe.Int64` doesn't get much use due to it being awkward to use). We might want to make it more consistent across targets before "promoting" it by making it as easy to create as a standard `Int` type.

## Alternatives

This could be done currently as a macro function but having to copy around a function into each project or pull in an external library to make creating a core numeric type easier isn't the best experience.

## Opening possibilities

There are a couple of other numeric types (`Single`, `UInt`) which are also under used and have inconsistent behaviour across targets, they could also gain suffixes in the future. Not sure if it's still planned but I think I remember hearing on one of the haxe livestreams that there were plans for more sized integer types, if this is the case more suffixes could be added to support those types as well.

Below is a table of possible suffix for intergers and floats and the potential types they would match to.

|suffix|integer size|
|--|--|
|i8|1 byte|
|i16|2 bytes|
|i32|4 bytes|
|i64|8 bytes|

|suffix|float size|
|--|--|
|f32|4 bytes|
|f64|8 bytes|

As an addition `u` could be prefixed to the integer suffixes to create unsigned versions (e.g. `ui64`, `ui8`).

## Unresolved questions

- Currently the compiler will error if you type a hexadecimal number with more than 8 characters (excluding `0x`), if we want hex literals to support the suffix should that be increased to 16 if suffixed?
