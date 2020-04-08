# Local static variables

* Proposal: [HXP-0000](0000-local-static.md)
* Author: [YellowAfterlife](https://github.com/yellowafterlife)

## Introduction

Provide syntax for defining static variables within the scope of a specific field.

## Motivation

Sometimes you want to reuse a static variable within a singular method -
perhaps it is a [data structure](https://github.com/YellowAfterlife/sfgml/blob/b7f32f37126d9ab9197d2248693c5a333019b86b/Array.hx#L242)
that you want to reuse to allocate an array/vector "just right",
or [a native function reference](https://github.com/HaxeFoundation/haxe/blob/c4c2d37f80c136e2259485c1d61b0bd8c38fecfe/std/neko/Lib.hx#L202)
that you resolve once on startup
or even just [depth for a recursive toString() call](https://github.com/HaxeFoundation/haxe/blob/c4c2d37f80c136e2259485c1d61b0bd8c38fecfe/std/cs/_std/Array.hx#L33).
Rest assured, it has to be static. And private. Not to be touched by anything else.

Currently most developers (and the standard library) take the approach of naming it something like `__<method>_<var>`,
but there's a limited amount of convenience in doing so.

-- todo --

## Detailed design

Describe the proposed design in details the way language user can understand
and compiler developer can implement. Show corner cases, provide usage examples,
describe how this solution is better than current workarounds.

## Impact on existing code

Currently `static` keyword is entirely forbidden inside method bodies so there should be no impact.

## Drawbacks

Describe the drawbacks of the proposed design worth consideration. This doesn't include
breaking changes, since that's described in the previous section.

## Alternatives

[As an old saying goes](https://yal.cc/wp-content/uploads/2019/03/haxe-macros.jpg),
most syntactic omissions can be corrected with a macro, and this is no exception,
but it would be preferred to not have to iterate the expression tree build-time
(as practice shows, this slowly adds up).

## Unresolved questions

Which parts of the design in question is still to be determined?
