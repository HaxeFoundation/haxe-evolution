# Module-level functions and variables

* Proposal: [HXP-0007](0007-module-level-funcs.md)
* Author: [Dan Korostelev](https://github.com/nadako)
* Status: [implemented in Haxe 4.2](https://github.com/HaxeFoundation/haxe/pull/8460)

## Introduction

Support defining `function`s and `var`s directly in the module (.hx file) instead of creating a class with static fields.

## Motivation

Classes are the heart of object-oriented programming found in languages like Java. In this paradigm, we model our program as interaction between class instances, so it makes sense to have classes as containers for everything. Non-factory static methods are relatively rare, so there's no practical need for functions defined outside of a class.

Nowadays, however, other programming paradigms, such as functional programming are becoming more and more popular, reducing the need for classic OOP classes and instead focusing more on functions that process passive data structures.

Haxe provides a lot of features supporting functional oriented paradigms (most importantly first-class functions), however it lacks a clean way to actually define functions without creating a wrapping class. This is annoying and gives a feeling of bloatedness to new people coming from non-OOP background. This is particularly unfortunate, because most of our target languages support plain functions, so having to wrap everything in a class can be a con when deciding whether to use Haxe or a target language directly.

## Detailed design

Supporting module-level functions should be pretty-straightforward. To minimize changes in compiler and its data structures, as well as the macro API, I propose the following:

 * add `TDFunction(name:String, fun:Function)` case to the `TypeDefKind` enum.
 * allow parsing functions at the module level and parse them into that `TDFunction`.
 * on module loading, when processing syntax declarations into module types, treat all module-level functions as `public static` methods of an implicitly created class. For this class we introduce a new `ClassKind` variant: `KModuleStatics` or something. This is very similar to how `KAbstractImpl`-classes are implicitly created for abstract types.
 * thereafter, when resolving an identifier (see more below), actually generate a static field access (`TTypeExpr(ModuleStatics).static(name)`).
 * when generating output, if target supports declaring plain functions (JavaScript, Lua, C++, etc.), a generator can decide to lose the wrapping `KModuleStatics` class and generate functions directly. If target requires a wrapping class (Java, C#) - generate like a normal class (plus, some optimizations can be applied, e.g. C# could mark class as `static`, and don't generate reflection helpers).

While the default access modifier for module-level functions is `public`, private module-level functions are supported by explicitly specifying the `private` keyword.

While this proposal describes module-level functions, vars are implemented the same with a `TDVar` variant. Properties can be supported too.

### Identifier resolution

The idea of module-level identifiers doesn't play particularly well with our current static field resolution mechanism, because we already have the concept of "primary module class" (a class with the same name as the module), so the most logical and least intrusive way to implement this would be to simply collect module-level functions in a implicitly created primary class and forbid explicit primary class declaration when there are module-level functions or vars.

Importing a module with `KModuleStatics` primary class should also imply `import ModuleName.*`, which means that all module-level functions/vars are imported by importing the module, which I believe would be the expected behaviour.

### Code example

Here's some code to break down the wall of text a bit. A slightly complicated hello-world command line script would look something like this:

Hello.hx
```haxe
inline var USAGE = "Usage: hello <name> <times>";

typedef Config = {
    name:String,
    times:Int,
}

function sayHello(config:Config) {
    for (i in 0...config.times)
        trace('Hi, ${config.name}!');
}

function main() {
    var args = Sys.args();
    if (args.length != 2)
        trace(USAGE);
    else
        sayHello({name: args[0], times: Std.parseInt(args[1])});
}
```

As you can see, mixing module-level functions and vars with other type declarations (`typedef Config` here) works just fine.

### Reflection

Since the module-level functions/vars end up being static class fields, the usual reflection should automatically work (e.g. `Reflect.field(Type.resolveClass("MyModule"), "myMethod")`). Actually I'm not sure if this is specified to work currently in Haxe, but if it does, generators should respect that and don't over-optimize `KModuleStatics` generation when reflection features are enabled.

### Macros, static extensions and so on

Since, again, module-level functions/vars are actually static fields, usual rules for `using` and `@:build(MyModule.myMethod)` calls apply, nothing new here.

### Final note

Because this document proposes implementing module-level functions in form of static class fields,
one may think that it would be inconsistent to have them public and imported implicitly with the module, remember however that this proposal is not about providing syntax sugar for static fields, but about defining functions and var on module level, and the fact they become static fields is an _implementation detail_.

Accessibility and resolution should follow those of other module-level declarations, that is: a module-level declaration (e.g. class/typedef/etc) is public unless specified as private, all module-level declarations are imported when the module is imported. Following these rules it makes sense to have module-levels functions to be public and be imported by default.

## Impact on existing code

With regard to existing code, this change can only potentially affect macro code because of newly introduced enum constructors in the macro API, and I believe that a very small portion of macro code will be affected by this, because it only matters for exhaustive pattern matches on `TypeDefKind` and `ClassKind` which are quite rare.

## Drawbacks

I don't immediately see any drawbacks in the proposed feature. On the contrary, I believe it'll make Haxe not only more competitive in terms of expressiveness in everyday use, but also easier to learn for absolute beginners in programming, because they won't have to learn the concept of a class and static methods from the start.

## Alternatives

I don't see any viable alternatives that would allow defining plain functions. Having a Haxe superset that is compiled to Haxe with a macro or in any other way isn't something anyone would seriously consider in practice.

## Opening possibilities

I think we could also use module-level functions to define extern functions, e.g.
```haxe
extern function SDL_Init(flags:UInt32):Int;
```

But that's gonna require some more thought, because it would mean that the implicitly created class must be made extern automatically.

## Unresolved questions

None so far.
