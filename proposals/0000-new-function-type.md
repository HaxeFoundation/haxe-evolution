# New function type syntax

* Proposal: [HXP-NNNN](0000-new-function-type.md)
* Author: [Dan Korostelev](https://github.com/nadako)

## Introduction

Provide a new, more natural syntax for declaring function types with support for argument names.

## Motivation

Haxe supports first-class functions from the beginning, using the following syntax for type-hinting variables containing functions:

```haxe
Int -> String -> Void
```

This syntax is often found in functional languages, such as OCaml and Haskell, but much less in non-functional and hybrid languages.

There are several issues with this syntax:

 * For people familiar with functional programming languages, it suggests that auto-currying and partial application are supported, but they aren't.
 * For non-FP people, it looks unfamiliar and differs too much from the actual function definition syntax.
 * It doesn't support argument names. While they aren't important for the type system, they are very useful for self-documenting code, IDE signature hints, callback auto-generation, etc. (see [fancy examples with screenshots here](https://github.com/HaxeFoundation/haxe/pull/6428#issue-239976019))

What we could have instead is a function type syntax that follows the new [arrow function](https://github.com/HaxeFoundation/haxe-evolution/blob/master/proposals/0002-arrow-functions.md) syntax:

```
(id:Int, name:String) -> Void
```

## Detailed design

### Syntax

The proposed syntax would looks like this:

```
// no arguments
() -> Void

// single argument
(name:String) -> Void

// multiple (also, optional) arguments
(name:String, ?age:Int) -> Void

// unnamed arguments
(Int, String) -> Bool

// mixed arguments, why not
(a:Int, ?String) -> Void
```

This is a rather small parser change, adding additional rules after the `(` token in `parse_complex_type` routine.

### Representation

There are different approaches regarding representation of named arguments in the AST:

 * Add a new `complex_type` variant: `CTNamed` for representing `name:Type` part and use that for `CTFunction` argument list. This would be similar to `CTOptional`.
 * Rework `CTFunction` arguments structure to contain a list of `(name, opt, type)` tuples, having empty string for names when the value comes from parsing unnamed arguments or old function type syntax.

Both options are viable and both will require changing macro data structures (which is breaking), so this is something we should discuss.

### Interoperability with the old syntax

Old syntax stays in place and works like before, examples are:

```haxe
Int -> Int
(Int) -> Int
(Int) -> Int -> Int
(Int -> Int) -> Int -> Int
// etc.
```

Mixing old and new syntax without parenthesis results in a syntax error:

```
(Int, Int) -> Int -> Int // syntax error: unexpected ->
(a:Int) -> Int -> Int // same
```

If the desired behaviour is to have a functional return type, parenthesis should be used:
```
(Int, Int) -> (Int -> Int)
(a:Int) -> (Int -> Int)
```

## Impact on existing code

Depending on how we implement AST data structures (options are listed in the [Representation](#representation) section), this will potentionally break macros that work with `haxe.macro.Expr.ComplexType` in one way or another, other than that it should not break anything because it's a completely new syntax.

## Drawbacks

The obvious drawback is that we'll have two function type syntaxes, which is why I think we should deprecate and eventually remove the old syntax. Removing the old syntax would be a huge change, but could still be an option for a major release, especially if we provide a migration tool.

## Alternatives

I already [tried the another approach](https://github.com/HaxeFoundation/haxe/pull/6428) of augmenting the current funtion type syntax with argument names: `a:Int -> b:String -> Void`, but unfortunately that introduced a [syntax ambiguity](https://github.com/HaxeFoundation/haxe/issues/6433) with macro reification and case patterns. One workaround for that would be to require parenthesis for named arguments, so it would be `(name:Type) -> Ret`, but that [suggests](https://github.com/HaxeFoundation/haxe/pull/6428#issuecomment-312671102) that one could do `(name:Type, name:Type) -> Ret` while it's an invalid syntax (one would have to do `(name:Type) -> (name:Type) -> Ret` instead).

## Unresolved questions

 * `ComplexType` representation for named arguments
 * Removal of old function type syntax
