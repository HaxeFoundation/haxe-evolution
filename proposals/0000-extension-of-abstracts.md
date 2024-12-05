# Extension of Abstracts

* Proposal: [HXP-0000](0000-extension-of-abstracts.md)
* Author: [EliteMasterEric](https://github.com/EliteMasterEric)

## Introduction

Provide the ability for classes to extend abstracts, which would cause them to extend the underlying type and then grant access to the abstract's types.

## Motivation

Say a library has a complicated internal class (let's say it's an extern to a native library, for example), which it then wraps in an abstract, like so:

```haxe
package lib.foo.Foo;

@:native("lib.foo.Foo")
extern class Foo_Inner { }

abstract Foo(Foo_Inner) {}
```

If an end user of this library wishes to extend Foo, they can either:

- Attempt to extends `Foo` itself, which gives a compilation error `Should extend using a class`
- Copy `Foo`, which results in duplicated code
- Extend `Foo_Inner`, which would require the user to re-implement the functionality of Foo.
- Add new functions to `Foo_Inner` itself, which would require all new functions added this way be added as `extern inline` functions.
- Extend `Foo_Inner`, then create an abstract of the new type, which again results in code duplication.

You can also see that if any class in the native library attempts to extend `Foo`, it will be unable to compile. However, if the classes are renamed (for example `Foo` to `FooWrapper` and `Foo_Inner` to `Foo`) to make this build, this mutates the class names of the API, and now end users who ware experienced with the native library are forced to change their code from referencing `Foo` (which they are used to) to `FooWrapper` (the new, compatible name).

## Detailed design

The behavior of the compiler would be modified such as to allow extension of abstracts.

- Any classes extending an abstract of a class will instead extend the underlying class.
  - This should also work for abstracts of abstracts of classes, abstracts of typedefs of classes, etc.
- Classes may only extend an abstract of a class.
  - Attempting to extend an abstract of an interface or anonymous structure should result in a compilation error.
- Instances of classes extending an abstract will be treated as instances of that abstract in Haxe code.
  - Instances of classes extending an abstract can be used as a variable of that abstract.
  - Instances of classes extending an abstract will be able to call any of that abstract's instanced methods.
- Calls to instance methods of the abstract will be translated to calls to the implementation class's static methods, the same as an abstract class.
- Classes extending an abstract of a class can access the private methods and variables of the "actual" parent class as though it was extending it.
- Classes may override functions of the parent class, but not functions of the abstract.
  - Attempting to override a function defined in the extended abstract should throw a compilation error.
- Interfaces can extend abstracts of interfaces the same as described above.
  - Classes should be able to implement abstracts of interfaces.
  - Classes should not be able to extend abstracts of interfaces, and interfaces should not be able to extend abstracts of classes.

```haxe
class Foo {
    public function new() {}

    public function value():Int { return 1; }
}

abstract Bar(Foo) {
    public function new() {
        this = new Foo();
    }

    // Calls to this function look like:
    // Bar_Impl_.doubleValue(foo_inst);
    public function doubleValue():Int {
        return 2 * this.value();
    }
}

// Before, this would throw a compilation error.
// With this change, Baz will generate as:
// class Baz extends Foo
class Baz extends Bar {
    public function new() {
        super();
    }

    // This function's body should look like:
    // return 2 * Bar_Impl_.doubleValue(this) + this.value();
    public function quintupleValue():Int {
        return 2 * this.doubleValue() + this.value();
    }

    // This should work as expected.
    public override function value():Int { return 3; }

    // This should throw a compilation error:
    // Cannot override a function declared in a parent abstract
    public override function doubleValue():Int;
}
```

## Impact on existing code

This would have no impact on existing code, as no existing code contains classes extending abstracts.

## Drawbacks

End users may be slightly confused about the behavior of abstracts when extending, succesfully calling and overriding some functions, then being unable to override others.

## Alternatives

As stated previously in the Motivation section, being unable to extend abstracts can be somewhat worked around by code duplication (not ideal) or mutating a library's API (also not ideal).

## Unresolved questions

I am looking for feedback as to edge cases this proposal does not cover.
