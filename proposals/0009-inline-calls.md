# Inlining functions at call location

* Proposal: [HXP-0009](0009-inline-calls.md)
* Author: [YellowAfterlife](https://github.com/yellowafterlife)
* Status: implemented in 4.0.0

## Introduction

Provide new syntax for requesting functions to be inlined at call location rather than place of definition.

## Motivation

This is a pretty common thing, even for [standard library](https://github.com/HaxeFoundation/haxe/blob/development/std/haxe/io/Input.hx#L229-L247) - you have a pair of functions of similar purpose and one does what other does plus a little extra (like handling sign bit in that case). Or have functions that are actively used inside of other functions (potentially benefiting from being inline there) while being used normally from "external" code.

But you cannot - a function is either inline (hinted or forced with @:extern) or it isn't.

## Detailed design

Considering the current syntax, I think it would be fitting to have this as `inline <TCall>` prefix "operator" - so this would permit to write the earlier shown code like
```haxe
public function readInt16() : Int {
	var n = inline readUInt16();
	if( n & 0x8000 != 0 ) ...
```
If the function cannot be inlined, an error could be shown as it is with `@:extern inline`.

## Impact on existing code

`inline` is not currently used in expression syntax at all and match rule is pretty clear so there shouldn't be any unexpected behaviour.

## Drawbacks

Documentation would need to note that the function inlined would be the one observed by the compiler/`$type`. This is fairly obvious and is an advantage (as you can have a base class methods share code without worrying about things getting strange if shared function is later overriden in a child class), but to be sure...

## Alternatives

Currently the best you can do is making a separate "this one function but inline" function, then have the "normal" version to call the inline version, and have other functions call either normal or inline version.

## Opening possibilities

If later allowing to use this with instantiation (TNew), the combination of two could be used to inline small "normal" classes in places of intensive use without having to deal with garbage collection or pooling. This doesn't have alternatives aside of duplicating classes or some [delightful hacks](https://try.haxe.org/#Ee0DC) (having a function to prevent inlining).
