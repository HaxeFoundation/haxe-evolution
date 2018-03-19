# A Const<T> Immutable Wrapper Type

* Proposal: [HXP-NNNN](NNNN-immutable-wrappers.md)
* Author: [Ben Morris](https://github.com/bendmorris)

## Introduction

Provide compile-time support for wrappers of values with recursively restricted mutability.

## Motivation

As an object-oriented language, Haxe values are mutable by default and any function may have side effects. Haxe also supports functional paradigms; in functional programming, immutable values and pure functions are used to limit and contain code with side effects. Supporting this in Haxe would enable safer code, capable of making guarantees about which parts of a program may contain side effects; it may also enable new compile-time optimizations.

Example use cases:

- An API that returns an internal array that shouldn't be modified. Rather than copy it, return it and hope for no modification, or write a custom abstract wrapper, with this proposal you can instead easily return a `Const<Array<T>>`.
- A backend API can provide a read-only value to the presentation layer, which can use it but not modify it.

## Detailed design

### The Const wrapper

Compiler support for a new wrapper type, `abstract Const<T>(T) from T` is proposed. `Const<T>` is a wrapper which enforces recursive immutability of the underlying value. `Const<T>` is a compile-time-only feature; at runtime the value is indistinguishable from a value of its underlying type.

A `Const<T>` value is subject to the following limitations:

- `T` can be implicitly coerced to a `Const<T>`. The reverse is not possible without `untyped`. An exception is basic types which are already immutable (Int, Float, enums...) which *can* be used interchangeably with `Const<T>` for ease of use.
- For an abstract or object, methods which are not marked with `@:const` (see below) may not be called.
- Fields may not be set. Properties can be get or set as long as the getter/setter is itself a `@:const` function.
- Field access will always return a `Const<U>` instead of their original type `U`, except for (1) basic types which are already immutable, or (2) if `U` itself is a `Const` type. (`@:const` methods do *not* return `Const<U>` by default; they return their original return type.)

Violations of these rules will result in a compile-time error.

For an `Const<Array<T>>`, this means:

```haxe
extern class Array<T> {
    // ...

    // indexOf can be called from a Const<Array>; its argument may also be a Const<T>
    @:const function indexOf( x : Const<T>, ?fromIndex:Int ) : Int;

    // push is not allowed from a Const<Array>
    function push(x:T):Int;

    // concat is safe to execute on an immutable array, and its argument
    // can also be immutable since neither array is modified.
    // If called on a Const<Array>, the returned Array will be mutable, but
    // its contents will not.
    @:const(:Array<Const<T>>) function concat(a:Const<Array<T>>):Array<T>;

    // ...
}
```

```haxe
    var a:Array<Int> = [1,2,3];

    // OK
    var b:Const<Array<Int>> = a;

    // OK, returns an Int (since Const<Int> is equivalent to Int)
    trace(b.length);

    // OK, because concat has @:const
    var c = b.concat([4,5,6]);

    // OK, because iterator() has @:const
    for (i in b) { // ... }

    // compile-time error: can't call non-@:const method from Const value
    b.push(4);

    // compile-time error: can't coerce Const<Array<Int>> back into Array<Int>
    var d:Array<Int> = b;

    // OK; explicitly sidestepping type system checks
    var e:Array<Int> = untyped b;
```

### Denoting immutability in Haxe code

Method and function calls are assumed to require a mutable (non-`Const`) value unless otherwise specified. To make `Const<T>` useful, we can use the `@:const` metadata to mark methods as being side-effect free, and allow these methods from an immutable value.

Some immutable methods (Array.concat) should return the same type regardless of whether they're called on a Const or a regular value, but others (iterator, array access, getters) should wrap their return value if they were called on a Const. For this reason the `@:const` metadata can take a type as an argument; when the read only version of the method is called, the result will be cast to the specified type. (Note that this uses a new `EComplexType` syntax as metadata arguments which has not yet been accepted or merged.)

```haxe
extern class Array<T> {
    // return value is always an Int
    @:const function indexOf( x : Const<T>, ?fromIndex:Int ) : Int;

    // iterating over a Const yields an iterator of Consts, protecting the
    // underlying array contents
    @:const(:Iterator<Const<T>>) function iterator() : Iterator<T>;
}
```

Function arguments can be typed as accepting `Const<T>` in order to denote that their arguments can be `Const<T>` wrappers. Since `T` unifies with `Const<T>` this will work with const or non-const arguments.

## Impact on existing code

No negative impact; this is a new, opt-in feature.

## Drawbacks

A method or expression could be incorrectly labeled as `@:const`. This information is relied on for optimizations, so this could be a problem, and it limits the guarantee of side-effect-free code.

This requires maintaining mutability as part of the type signature in the standard library, which shouldn't be too much additional effort and may have benefits for static analysis.

## Alternatives

This functionality can be mimicked by using a custom `@:genericBuild` macro. However, this approach is less powerful and has a number of drawbacks:

- A parameterized `Const<T>` in a generic is impossible; T must be known for the macro to generate the Const wrapper.
- `Const<T>` must be a class. An abstract is impossible because the underlying type is unknown and unconstrained.
- It's difficult to support this for method calls without a mechanism to either explicitly mark methods as immutable, or having the compiler infer from the method body. This makes wrappers for builtin types like Array difficult to implement.
- Potential slowdown due to a large number of generated types.

## Opening possibilities

In addition to safer code and denoting where side effects can happen, adding information about mutability may enable static analyzer improvements, such as automatic inlining of constant expressions or elimination of function calls with unused return values.

`@:const` methods or expressions may be allowed as initial values for static variables.

## Unresolved questions

How to handle `@:const`-ness in interfaces or overridden methods? (We should probably enforce that they're the same, as if they're part of the type signature.)

Is it feasible for the compiler to try to infer `@:const`-ness of a method body?

In some cases it may be necessary to use `untyped` to convert a Const back to its original value (for example in a method which accepts `Const<T>` as an argument; Neko's Array.concat was one case where this was necessary.) In my opinion this is okay and shouldn't be feared. `untyped` removes the type system's ability to make correctness guarantees, which is okay as long as its use is self-contained and easy enough to reason about so we can make those guarantees ourselves.
