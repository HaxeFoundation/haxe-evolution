# A Const<T> Immutable Wrapper Type

* Proposal: [HXP-NNNN](NNNN-immutable-wrappers.md)
* Author: [Ben Morris](https://github.com/bendmorris)

## Introduction

Provide compile-time support for limiting mutability of values and expressing immutability of expressions/functions.

## Motivation

As an object-oriented language, Haxe values are mutable by default and any function may have side effects. Haxe also supports functional paradigms; in functional programming, immutable values and pure functions are used to limit and contain code with side effects. Supporting this in Haxe would enable safer code, capable of making guarantees about which parts of a program may contain side effects; it may also enable new compile-time optimizations.

## Detailed design

### The Const wrapper

Compiler support for a new wrapper type, `abstract Const<T>(T) from T` is proposed.

`Const<T>` is a compile-time-only feature. At runtime the value is indistinguishable from a value of its underlying type.

A `Const<T>` value is subject to the following limitations:

- `T` can always be implicitly coerced to a `Const<T>`. The reverse is not possible without `untyped`. Const wrappers of existing values can also be explicitly created via a constructor: `new Const(value);`
- For an abstract or object, methods which are not marked with `@:const` (see below) may not be called.
- Fields/properties may not be set, unless the property has a setter which is itself a `@:const` function. (Properties cannot be accessed if their getter is not a `@:const` function.)
- Fields will always return a `Const<U>` instead of their original type `U`, except for (1) basic types which are already immutable, or (2) if `U` itself is a `Const` type. (Function return values do *not* need to be transformed to return `Const<U>`.)

Violations of these rules will result in a compile-time error.

For an `Const<Array<T>>`, this means:

```haxe
extern class Array<T> {
    // ...

    // concat is safe to execute on an immutable array, and its argument
    // can also be immutable since neither array is modified
    @:const function concat(a:Const<Array<T>>):Array<T>;

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

In cases where `T` is already an immutable value (Int, Float, Bool, String, or an Enum instance) `Const<T>` is equivalent to `T`.

### Denoting immutability in Haxe code

Method and function calls are assumed to require a mutable (non-`Const`) value unless otherwise specified. To make `Const<T>` useful, we need a standardized way to mark methods or expressions as being side-effect free, such that they can be used with an immutable value. One simple option is through use of a `@:const` metadata. `@:const` could be used on either an expression or a function definition:

- Instance or abstract methods with this metadata are safe to be called via `Const<T>` wrappers.
- Expressions which have this metadata and interact with `Const<T>` wrappers will be considered safe usage.

Function arguments can be typed as accepting `Const<T>` in order to denote that their arguments can be `Const<T>` wrappers.

## Impact on existing code

No negative impact; this is a new, opt-in feature.

## Drawbacks

A method or expression could be incorrectly labeled as `@:const`. If this information is relied on for optimizations, this could be a problem; it can also break the guarantee of side-effect-free code.

## Alternatives

This functionality can be mimicked by using a custom `@:genericBuild` macro. However, this approach is less powerful and has a number of drawbacks:

- A parameterized `Const<T>` in a generic is impossible; T must be known for the macro to generate the Const wrapper.
- `Const<T>` must be a class. An abstract is impossible because the underlying type is unknown and unconstrained.
- It's difficult to support this for method calls without a mechanism to either explicitly mark methods as immutable, or having the compiler infer from the method body. This makes wrappers for builtin types like Array difficult to implement.
- Potential slowdown due to a large number of generated types.

## Opening possibilities

Adding information about mutability may enable static analyzer improvements, such as automatic inlining of constant expressions or elimination of function calls with unused return values.

`@:const` methods or expressions may be allowed as initial values for static variables.

## Unresolved questions

How to handle `@:const`-ness in interfaces or overridden methods?

Is it feasible for the compiler to try to infer `@:const`-ness of an expression?
