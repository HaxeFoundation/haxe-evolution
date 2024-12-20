# Structured Constructors

* Proposal: [HXP-0020](0020-structured-constructors.md)
* Author: [0b1kn00b](https://github.com/ohmrun)

## Introduction

It's possible to largely automate constructors with no additions to the AST, this would 
decrease boilerplate, as well as open up possibilities for longer chains of inference
in abstract types and less visual noise in type transformations. 

## Motivation

It's not necessary for everything to have a complete order. Pointers are not predictably ordered, for example, but as practically everthing is a composit, *something* will likely have an order that can be used to produce sufficient order that the total will be deterministic.

This information could be used to build constuctors in the analysis phase.

## Detailed design

A constructor argument set can be modelled as a regular Haxe `abstract` over an `enum`, using the abstract `@:from` to pattern match. 
The thinking for this is like the class implementation of abstract, there's a predictable, ordered pattern which can be used to create an intermediate structure which is transparent to the type system and AST.

Julialang has an interesting implementation of this with parts of the DataType DataType (irc)

I have examples of this [here](https://github.com/ohmrun/fletcher/blob/develop/src/main/haxe/eu/ohmrun/fletcher/Modulate.hx)

You need an `EnumValue` for each constructor and an `abstract` over that for more cleverness

These strutures can largely be infered through a shortlex algorithm over the elements of the typesystem as it stands.

## Impact on existing code

A `constructors` `abstract` over an `EnumType` on `haxe.macro.BaseType` might break macro code, although if it's made optional where the feature isn't used the breakages would be minimal. 

## Drawbacks

It might make macro code more complex, as enums are one of the more complex structures to transform.

## Alternatives

You can get around this with boilerplate and macros, but macros being one level deep makes problems with composition across libraries.

## Opening possibilities
## Unresolved questions

Do we overload new? a new keyword? a metadata annotation? I think it can be done in a backward compatible way. 