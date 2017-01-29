# Immutable type modifier

* Proposal: [HXP-NNNN](NNNN-immutable.md)
* Author: [Dan Korostelev](https://github.com/nadako)

## Introduction

Support immutable type modifier (e.g. `const`) to provide compile-time checked read-only access to objects of any type.

## Motivation

Often it's required to ensure read-only access to data in some parts of a program so one can't mistakenly change
what's not supposed to be changed.

For example, some view component should only be allowed to read data from a given object and present it to the user,
but shouldn't ever change it.

Or a "data storage" system where you can retrieve a structure and read its values, but to change them you must use
the "set" method that maintains a transaction. Changing the structure directly, while allowed by type system will
lead to bugs in this scenario.

In general one can think of many cases where only specific part of a program is allowed to change the state while others
must only read it. Moreover, in a pure functional style code any access is read-only and state is never modified.

Currently there are several ways to provide such safety, but none of them are good enough or universal, let's look at
some of them:

 * copying - we can pass a copy of the object every time we need to ensure the original one won't be changed,
   but this obviously has a significant performance cost and loses the identity equality between objects, so it's not
   really an answer to immutability.
 * read-only wrapper types - we can define a read-only wrapper type that implicitly casts from the original type,
   for example - a new structure type with same fields but `(default,never)` access, or an abstract for a class with
   `(get,never)` properties that forward read access to class fields. To provide recursive immutability, one has to
   define read-only types for every field type too which results in a ton of additional types with different names.
   One can use macro to automate that, but some issues can't be solved by it, as well as new issues are being introduced.
   Here I'll just copy my [comment](https://github.com/HaxeFoundation/haxe-evolution/issues/17#issuecomment-275680691)
   from the other immutability discussion:
   
   >> Would not a haxe.Immutable<{ field : Int }> be enough? (we can use a macro to implement it)
   >
   > I'm using such a macro (`@:genericBuild(...) class Const<T>`) in my work project, which seemed to be a great idea at first, but it has quite a list of problems, some of them are (in no particular order):
   > * the macro slowdown is VERY significant when you use a lof of those types
   > * you can't do `typedef A<T> = { foo:Const<T> }`, since a macro won't know what to build from that
   > * generated type name looks somewhat like `Const_original_pack_OriginalName` or `const.original.pack.OriginalName` which is quite ugly and can be confusing in autocompletion
   > * generated type/module depends on the module the macro was called from which screws up compilation/completion server (I still have no idea how to fix that properly in my codebase)
   > * type parameter invariance creates obstacles when you try to use generated constant types in type parameters (for collections) which can be only worked around by using `cast`
   > * you can't generate a `Const` version of an enum that will be compatible with the original (since it'll be a completely different type)
   >
   > There are probably more of them which I can't remember right now, but these alone are enough to agree with @fponticelli that "macro suck for these kind of things" :) So I'm really in favor of proper compiler support for that.

---

What I'd like to propose instead is to introduce a type modifier, like `const` that will instruct compiler that this type
instance is recursively constant, meaning that its fields can't be changed on any level of hierarchy. This modifier should
would work for any type, including classes, anonymous structures, abstracts and enums (so their params are typed as const).

## Detailed design

Ideally, we introduce a new keyword, like `const` as in the above that is used to specify that
this usage of a type should be treated as immutable. This keyword will also be used as a method
access modifier to specify that `this` is typed as immutable within the method, also meaning that
this method is safe to call on immutable versions of a class.

Let's look at the example of how code using this fature would look like:

```haxe
class User {
  var name:String;
  var bestFriend:User;

  static function printUser(user: const User) { // could also be const(User) or something...
    // read access allowed
    trace(user.name);
    trace(user.bestFriend.name);

    // ERROR: write access not allowed
    user.name = "Duh";
    user.bestFriend = null;

    // ERROR: write access not allowed,
    // because `user.bestFriend` is also typed as `const User`
    user.bestFriend.name = "Me";
    $type(user.bestFriend); // const User
  }

  const function isOwnBestFriend() {
    $type(this); // const User
    this.name = "Duh"; // ERROR: write access not allowed
    return this == bestFriend;
  }
}
```

So, basically when type is marked as `const`, the following perks apply to it:
 * field write access is forbidden
 * field read access expression types are also marked as `const` (so immutability is recursive)
 * calling non-const methods of that type is forbidden

When method is marked as `const`, within its body `this` is typed as `const`, so the rules above are applied
to the usage of the object from that method.

Unification rules are the following:

 * `T` can be assigned to `const T`
 * `const T` can NOT be assigned to `T`
 * `{a: T}` can be assigned to `{a: const T}`
 * `{a: const T}` can NOT be assigned to `{a: T}`
 * `Array<T>` can be assigned to `Array<const T>`
 * `Array<const T>` can NOT be assigned to `Array<T>`

It should also work for enum instances by adding `const` to the constructor arguments, for example:

```
enum E {
  A;
  B(obj:{f:Int});
}

function f(e: const E) {
  switch (e) {
    case A:
    case B(obj):
      obj.f = 15; // ERROR: write access not allowed
  }
}
```


## Impact on existing code

This is a new feature, so it shouldn't break a lot of exiting code, however depending on how it's implemented,
there are two things to consider:

 * introducing a new keyword always can break code, though it's doubtful that the proposed `const` identifier
   is used a lot, considering that it's a reserved keyword in many popular languages

 * introducing immutability on syntax and type system level will require additions to macro-related data
   structures, such as `haxe.macro.ComplexType` and `haxe.macro.Type`, which can potentially break macro code

## Drawbacks

TODO: are there any?

## Alternatives

A more radical alternative would be to make all types immutable by default and require explicit mutability,
which is more FP-friendly, but is too breaking and not in the line with Haxe's general direction.

## Opening possibilities

Introducing `const` keyword would also allow to use it for defining immutable local name (as opposed to `var`),
but that's out of the scope of this proposal.

## Unresolved questions

Which parts of the design in question is still to be determined?
