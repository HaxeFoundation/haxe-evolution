# A Const<T> Immutable Wrapper Type

* Proposal: [HXP-NNNN](NNNN-immutable-wrappers.md)
* Author: [Ben Morris](https://github.com/bendmorris)

## Introduction

Provide compile-time support for wrappers of values with recursively restricted mutability.

## Motivation

As an object-oriented language, Haxe values are mutable by default and any function may have side effects. Haxe also supports functional paradigms; in functional programming, immutable values and pure functions are used to limit and contain code with side effects. Supporting this in Haxe would enable safer code, capable of making guarantees about which parts of a program may contain side effects; it may also enable new compile-time optimizations.



## Detailed design

### The Const wrapper

Compiler support for a new wrapper type, `abstract Const<T>(T) from T` is proposed. `Const<T>` is a wrapper which enforces recursive immutability of the underlying value. `Const<T>` is a compile-time-only feature; at runtime the value is indistinguishable from a value of its underlying type.

A `Const<T>` value is subject to the following limitations:

- `T` can be implicitly coerced to a `Const<T>`. The reverse is not possible without `untyped`. An exception is basic types which are already immutable (Int, Float, enums...) which *can* be used interchangeably with `Const<T>` for ease of use.
- For an abstract or object, methods which are not marked with `@:pure` (see below) may not be called.
- Fields may not be set. Properties can be get or set as long as the getter/setter is itself a `@:pure` function.
- Field access will always return a `Const<U>` instead of their original type `U`, except for (1) basic types which are already immutable, or (2) if `U` itself is a `Const` type. (`@:pure` methods do *not* return `Const<U>` by default; they return their original return type.)

Violations of these rules will result in a compile-time error.

For an `Const<Array<T>>`, this means:

```haxe
extern class Array<T> {
    // ...

    // concat is safe to execute on an immutable array, and its argument
    // can also be immutable since neither array is modified
    @:pure function concat(a:Const<Array<T>>):Array<T>;

    // push on the other hand is not safe from an immutable array
    function push(x:T):Int;

    // ...
}
```

- The `Array<T>` can be coerced to a `Const<Array<T>`: `var c:Const<Array<Int>> = [1];` or `var c = Const([1]);`
- `concat` may be called on a `Const<Array<T>>`, returning a (mutable) `Array<T>`.
- `length` may be accessed on a `Const<Array<T>>`, returning a `Const<Int>` (which has no difference from a regular Int.)
- Iterating over or indexing the array will produce `Const<T>` values.
- An attempt to call `push` will throw a compiler error.

```haxe
    var a:Array<Int> = [1,2,3];

    // OK
    var b:Const<Array<Int>> = a;

    // OK, returns an Int (since Const<Int> is equivalent to Int)
    trace(b.length);

    // OK, because concat has @:pure
    var c = b.concat([4,5,6]);

    // OK, because iterator() has @:pure
    for (i in b) { // ... }

    // compile-time error: can't call non-@:pure method from Const value
    b.push(4);

    // compile-time error: can't cast Const<Array<Int>> back into Array<Int>
    var d:Array<Int> = b;

    // OK; explicitly sidestepping type system checks
    var e:Array<Int> = untyped b;
```

### Denoting immutability in Haxe code

Method and function calls are assumed to require a mutable (non-`Const`) value unless otherwise specified. To make `Const<T>` useful, we can leverage the existing `@:pure` metadata to mark methods as being side-effect free, and allow these methods from an immutable value.

Function arguments can be typed as accepting `Const<T>` in order to denote that their arguments can be `Const<T>` wrappers. Since `T` unifies with `Const<T>` this will work with const or non-const arguments.

## Impact on existing code

No negative impact; this is a new, opt-in feature.

## Drawbacks

A method or expression could be incorrectly labeled as `@:pure`. If this information is relied on for optimizations, this could be a problem; it can also break the guarantee of side-effect-free code.

## Alternatives

This functionality can be mimicked by using a custom `@:genericBuild` macro. However, this approach is less powerful and has a number of drawbacks:

- A parameterized `Const<T>` in a generic is impossible; T must be known for the macro to generate the Const wrapper.
- `Const<T>` must be a class. An abstract is impossible because the underlying type is unknown and unconstrained.
- It's difficult to support this for method calls without a mechanism to either explicitly mark methods as immutable, or having the compiler infer from the method body. This makes wrappers for builtin types like Array difficult to implement.
- Potential slowdown due to a large number of generated types.

## Opening possibilities

In addition to safer code and denoting where side effects can happen, adding information about mutability may enable static analyzer improvements, such as automatic inlining of constant expressions or elimination of function calls with unused return values.

`@:pure` methods or expressions may be allowed as initial values for static variables.

## Unresolved questions

How to handle `@:pure`-ness in interfaces or overridden methods? (We should probably enforce that they're the same, as if they're part of the type signature.)

Is it feasible for the compiler to try to infer `@:pure`-ness of a method body?

There should probably be a way for methods to auto-"constify" their return value as field access does, i.e. you can call this method on a value or a Const, but if called on the Const, the return value is a Const too. This would be useful for getters, array access, iterators...

In some cases it may be necessary to use `untyped` to convert a Const back to its original value (for example in a method which accepts `Const<T>` as an argument; Neko's Array.concat was one case where this was necessary.) In my opinion this is okay and shouldn't be feared. `untyped` removes the type system's ability to make correctness guarantees, which is okay as long as its use is self-contained and easy enough to reason about so we can make those guarantees ourselves.
