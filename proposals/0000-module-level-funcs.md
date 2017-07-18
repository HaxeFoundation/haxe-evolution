# Module-level functions and variables

* Proposal: [HXP-NNNN](NNNN-module-level-funcs.md)
* Author: [Dan Korostelev](https://github.com/nadako)

## Introduction

Support defining functions directly in the module (.hx file) instead
of creating a class with static methods.

## Motivation

Classes are the heart of object-oriented programming found in languages like Java. In this paradigm, we
model our program as interaction between class instances, so it makes sense to have classes as containers
for everything. Non-factory static methods are relatively rare, so there's no need for functions defined outside of a class.

Nowadays, however, other programming paradigms, such as functional programming is becoming more and more
popular, reducing the need for classic OOP classes and instead focusing on functions that process passive
data structures.

Haxe provides a lot of features supporting functional oriented paradigms (most importantly first-class functions),
however it lacks a clean way to actually define functions without creating a wrapping class. This is annoying and
gives a feeling of bloatedness to new people coming from non-OOP background. This is particularly unfortunate,
because most of our target languages support plain functions, so having to wrap everything in a
class can be a con when deciding whether to use Haxe or a target language directly.

## Detailed design

Supporting module-level functions should be pretty-straightforward. To minimize changes in compiler and its data
structures, as well as the macro API, I propose the following:

 * add `TDFunction(name:String, fun:Function)` case to the `TypeDefKind` enum;
 * allow parsing functions at the module level and parse them into that `TDFunction`;
 * when processind syntax into typed AST, treat all module-level functions as static methods of an
   implicitly created class. For this class we introduce a new `ClassKind` variant: `KModuleStatics` or
   something. This is very similar to how `KAbstractImpl`-classes are implicitly created for abstract types.
 * thereafter, when resolving an identifier (see more below), actually generate a static field access (`TTypeExpr(ModuleStatics).static(name)`)
 * when generating output, if target supports declaring plain functions (JavaScript, Lua, C, etc.), a generator
   can decide to lose the `KModuleStatics` class and generate functions directly. If target requires a wrapping
   class (Java, C#), generate like a normal class (plus, some optimizations an be applied, e.g. C# could mark
   class as `static`, and don't generate reflection helpers).

While this proposal describes module-level functions, I think it would be logical and consistent (although less useful) to also allow module-level variables, treating them similarly static vars.

The `-main` argument should also handle module-level functions for program entry points.

### Identifier resolution

The idea of module-level identifiers doesn't play particularly well with our current static field resolution mechanism, mainly because we already have the concept of "primary module class", so we have to think about the sane resolution rules. Here's what I propose:

 * `import MyModule` imports both types and module-level functions from `MyModule` to local namespace.
 * `import MyModule.method` if there's a module-level `method` function - import it, otherwise look for `method` in the primary class statics as it's done now.
 * `import MyModule.*` imports both module-level functions and primary class statics, module-levels taking precedence.

fully-qualified paths are resolved accordingly:

 * `MyModule.method()` calls the module-level function, if found, otherwise calls static method of the primary class.

> TODO: Maybe we should just forbid having explicit primary type if there are module-level functions and generate
it implicitly. This should cover most use-cases and greatly simplify resolution rules (basically, no changes to it will be required at all).

## Impact on existing code

> TODO: only macro data structures, not changed but added, might only break exhaustive pattern matches.

## Drawbacks

> TODO: Describe the drawbacks of the proposed design worth consideration. This doesn't include breaking changes, since that's described in the previous section.

## Alternatives

> TODO: What alternatives have you considered to address the same problem, why the proposed solution is better?

## Opening possibilities

> TODO: Does this change make other future changes possible or easier? Leave this section out if the proposed change
is completely self-contained.

## Unresolved questions

> TODO: Which parts of the design in question is still to be determined?
