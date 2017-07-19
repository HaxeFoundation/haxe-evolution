# Module-level functions

* Proposal: [HXP-NNNN](NNNN-module-level-funcs.md)
* Author: [Dan Korostelev](https://github.com/nadako)

## Introduction

Support defining functions directly in the module (.hx file) instead of creating a class with static methods.

## Motivation

Classes are the heart of object-oriented programming found in languages like Java. In this paradigm, we model our program as interaction between class instances, so it makes sense to have classes as containers for everything. Non-factory static methods are relatively rare, so there's no practical need for functions defined outside of a class.

Nowadays, however, other programming paradigms, such as functional programming are becoming more and more popular, reducing the need for classic OOP classes and instead focusing more on functions that process passive data structures.

Haxe provides a lot of features supporting functional oriented paradigms (most importantly first-class functions), however it lacks a clean way to actually define functions without creating a wrapping class. This is annoying and gives a feeling of bloatedness to new people coming from non-OOP background. This is particularly unfortunate, because most of our target languages support plain functions, so having to wrap everything in a class can be a con when deciding whether to use Haxe or a target language directly.

## Detailed design

Supporting module-level functions should be pretty-straightforward. To minimize changes in compiler and its data structures, as well as the macro API, I propose the following:

 * add `TDFunction(name:String, fun:Function)` case to the `TypeDefKind` enum.
 * allow parsing functions at the module level and parse them into that `TDFunction`.
 * on module loading, when processing syntax declarations into module types, treat all module-level functions as static methods of an
   implicitly created class. For this class we introduce a new `ClassKind` variant: `KModuleStatics` or
   something. This is very similar to how `KAbstractImpl`-classes are implicitly created for abstract types.
 * thereafter, when resolving an identifier (see more below), actually generate a static field access (`TTypeExpr(ModuleStatics).static(name)`).
 * when generating output, if target supports declaring plain functions (JavaScript, Lua, C++, etc.), a generator can decide to lose the `KModuleStatics` class and generate functions directly. If target requires a wrapping class (Java, C#) - generate like a normal class (plus, some optimizations an be applied, e.g. C# could mark class as `static`, and don't generate reflection helpers).

> While this proposal describes module-level functions, I think it would be logical and consistent (although less useful) to also allow module-level variables, treating them similarly static vars.

The `-main` argument should also handle module-level functions for program entry points.

### Identifier resolution

The idea of module-level identifiers doesn't play particularly well with our current static field resolution mechanism, mainly because we already have the concept of "primary module class" (a class with the same name as the module), so we have to think about the sane resolution rules.

We have two options to deal with this:

 1) Implicitly create primary module class for module-level functions and forbid explicit primary module class in this case. This would be the safest and most minimal change.
 2) Allow both primary module class and module statics class and apply additional identifier resolution rules:
    * `import MyModule` imports both types and module-level functions from `MyModule` to local namespace.
    * `import MyModule.method` if there's a module-level `method` function - import it, otherwise look for `method` in the primary class statics as it's done now.
    * `import MyModule.*` imports both module-level functions and primary class statics, module-levels taking precedence.
    * `MyModule.method()` calls the module-level function, if found, otherwise calls static method of the primary class.

I propose going for the first option at least for now, because that would greatly simplify implementation and won't introduce new resolution semantics. Moreover, I think it'll be what people actually want when using module-level function declarations.

## Impact on existing code

With regard to existing code, this change can only potentially affect macro code because of newly introduced enum constructors in the macro API, and I believe that a very small portion of macro code will be affected by this, because it only matters for exhaustive pattern matches on `TypeDefKind` and `ClassKind` which are quite rare.

## Drawbacks

I don't immediately see any drawbacks in the proposed feature. On the contrary, I believe it'll make Haxe not only more competitive in terms of expressiveness in everyday use, but also easier to learn for absolute beginners in programming, because they won't have to learn the concept of a class and static methods from the start.

## Alternatives

I don't see any viable alternatives that would allow defining plain functions. Having a Haxe superset that is compiled to Haxe with a macro or in any other way isn't something anyone would seriously consider in practice.

## Unresolved questions

 * importing/identifier resolution - two variants proposed.
 * module-level vars - generally considered bad style, but would be consistent to have and can actually be useful for small scripts
