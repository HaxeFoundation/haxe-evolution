# New function type syntax

* Proposal: [HXP-NNNN](0000-new-function-type.md)
* Author: [Dan Korostelev](https://github.com/nadako)

## Introduction

Provide a new, more natural syntax for declaring function types with support for argument names.

## Motivation

Haxe supports first-class functions from the beginning, using the following syntax for type-hinting
variables containing functions:

```haxe
Int->String->Void
```

This syntax is often found in functional languages, such as OCaml and Haskell, but much less
in non-functional and hybrid languages.

There are several issues with this syntax:

 * For people familiar with functional programming languages, it suggests that auto-currying and partial application are supported, but they aren't.
 * For non-FP people, it looks unfamiliar and differs too much from the actual function definition syntax.
 * It doesn't support argument names. While they aren't important for the type system, they are very useful for self-documenting code, IDE signature hints, callback auto-generation, etc. (see [fancy examples with screenshots here](https://github.com/HaxeFoundation/haxe/pull/6428#issue-239976019))

What we could have instead is a function type syntax that somewhat follows the new [arrow function](https://github.com/HaxeFoundation/haxe-evolution/blob/master/proposals/0002-arrow-functions.md) syntax:

```
(id:Int, name:String)->Void
```

## Detailed design

The proposed syntax would looks like this:

```
// no arguments
() -> Void

// single argument
(name:String) -> Void

// multiple (also, optional) arguments
(name:String, ?age:Int) -> Void
```

Argument names have to be mandatory to distinguish it from the current function type syntax,
however if we deprecate and remove the current syntax someday, we could lift the restriction
and support type-only syntax as well:

```
(Int, String) -> Bool
```

> If you're wondering what's the problem with distinguishing the new syntax from the current one without mandatory argument names, it's the single-argument case like `(Int)->Int->Void`. This is currently interpreted like and `Int->Int->Void` function (two integer args, returning void),
however with the proposed syntax it should mean `(Int->Int)->Void` (a function taking a single-int-returning-int function and returning void). With mandatory argument names, we can parse
`(a:Int)->Int->Void` unambigously, because with the old syntax that would be a syntax error.

Implementation-wise, this is mainly a parser change, adding additional rules after the `(` token,
however there are different approaches regarding actual representation:

 * Add a new `complex_type` variant: `CTNamed` for representing `name:Type` part that's only allowed to be parsed within parentheses (to prevent `case macro : Type:` ambiguity) and use that for `CTFunction` argument list. This would be similar to `CTOptional`.
 * Rework `CTFunction` arguments structure to contain a list of `(name, opt, type)` tuples, having empty string for names when the value comes from parsing old function type syntax.

Both options are viable and both will require changing macro data structures (which is breaking), so this is something we should discuss.

## Impact on existing code

Because of the required function type structure changes, it will break macros that work
with `haxe.macro.Expr.ComplexType`, other than that it should not break anything because
it's a completely new syntax.

If we want to deprecate/remove the old syntax, so we can have one function type syntax (and
this support unnamed arguments as mentioned above), we can provide a migration tool based on
[hxparser](https://github.com/vshaxe/hxparser) that automatically changes old syntax to new,
e.g. `Int->String->Void` to `(a:Int, b:String)->Void`.

## Drawbacks

The obvious drawback is that we'll have two function type syntaxes, which is why I think we should
either deprecate or remove the old syntax. Removing the old syntax would be a huge change, but
we're talking about a major release, and we are able to provide an automatic migration tool.

## Alternatives

I already [tried the another approach](https://github.com/HaxeFoundation/haxe/pull/6428) of "augmenting" the current funtion type syntax with argument names: `a:Int -> b:String -> Void`,
but unfortunately that introduced a [syntax ambiguity](https://github.com/HaxeFoundation/haxe/issues/6433) with macro reification and case patterns. One workaround for that would be to require parenthesis for named arguments, so it would be `(name:Type)->Ret`, but that [suggests](https://github.com/HaxeFoundation/haxe/pull/6428#issuecomment-312671102) that one could do `(name:Type, name:Type)->Ret` while it's an invalid syntax.

## Unresolved questions

 * ComplexType representation for named arguments
 * Removal of old function type syntax
 * Support of unnamed arguments `(Int,Int)->Void`, if old function type syntax is removed
