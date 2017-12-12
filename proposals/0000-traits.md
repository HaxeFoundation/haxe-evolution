# Traits

* Proposal: [HXP-NNNN](NNNN-traits.md)
* Author: [Ben Morris](https://github.com/bendmorris)

## Introduction

A more powerful analog to static extension that enables open polymorphism, modular type specifications and wrapping interfaces and new behavior over existing types.

## Motivation

As implemented in [Rust](https://rustbyexample.com/trait.html), traits are a more powerful extension of interfaces from OOP, with some key differences:

- Implementing traits is external to the definition of the type; traits may be implemented for existing types, including basic types, outside of the original type declaration.
- Traits may include default implementations for methods.

Rust and Haxe share an emphasis on providing powerful, zero-runtime-cost abstractions. Haxe provides several analogous features for extending the functionality of existing types, including static extension and abstracts. However, none of these features is as powerful as traits; specifically, they lack the ability to define a single type which wraps multiple existing types. Adopting the Rust trait system for Haxe will open the door to new ways of defining behavior in a modular and extensible way.

Benefits of traits include:

- Providing an elegant way to express polymorphism over an open set of types, including types which are *not* classes such as primitives, abstracts or structures.
- Breaking up large class/abstract definitions into more concise blocks by functionality.
- Providing a standard way to extend language-level functionality (see "opening possibilities".)

Caveats to this proposal:

- Some OOP languages such as PHP also include a "trait" concept, also known as a "mixin", focused around code reuse. These are orthogonal concepts that unfortunately share a name. This proposal deals with the specific implementation of traits from Rust as a method of open polymorphism.
- Adding optional default implementations to interfaces (as supported in [tink_lang](https://github.com/haxetink/tink_lang) traits) is outside of the scope of this proposal. Because traits are implemented via interfaces here, default method implementations would be relatively straightforward to add in the future, which would also satisfy the "mixin" definition of a trait.

## Detailed design

To avoid duplication in the language, traits can simply reuse the existing interface concept:

```haxe
interface Serializable
{
	public function toString():String;
}
```

Traits would add two new capabilities to Haxe: (1) the ability to **implement interfaces outside of a class declaration**, and (2) the ability for **non-class types to implement interfaces**. This is done via the `implement` keyword:

```haxe
implement Serializable for MyClass
{
	public function toString():String return "MyClass " + this.id;
}

implement Serializable for Int
{
	public function toString():String return Std.string(this);
}

implement Serializable for {name: String}
{
	public function toString():String return this.name;
}
```

The following methods of implementing an interface are equivalent (but see Unresolved questions):

```haxe
interface MyInterface {}

// current way
class MyClass1 implements MyInterface {}

// trait way
class MyClass2 {}
implement MyInterface for MyClass2 {}
```

An `implement` block may only include the methods of the interface, and must include them all.

These underlying types will now unify with the interface:

```haxe
class Serializer
{
	public static function serialize(value:Serializable):String
	{
		return value.toString();
	}

	static function main()
	{
		trace(serialize(42));
		trace(serialize("banana"));
	}
}
```

Trait implementations could be compiled in two ways, similar to generics:

- When used as a generic type parameter, traits resolve fully at compile time, and will generate a separate version of the function per implementer:

```haxe
@:generic public static function serialize<T:Serializable>(value:T)
{
	return value.toString();
}
```

- Otherwise, implementation of a trait will generate an implementing class to which the type will be coerced at runtime to unify implementers with their traits; trait usage is more restricted in these cases to avoid conflicts.

Trait implementations must not conflict; conflicts will be treated as a compile-time error. Examples of disallowed combinations due to the possibility of ambiguity in choosing the trait implementation include:

- Two distinct interfaces which do not extend each other, since a class could implement both.
- Anonymous structures which each have unique fields, since a structure could contain both sets.
- `Null<T>` and `T`.
- An abstract and its underlying type for traits used outside of @:generic functions.

Multiple trait implementations are *not* in conflict if they are unambiguous, or if one is strictly "more specific" than the other. For example, the following are allowed:

- A parent and child class; the child takes precedence.
- An parent interface and the interface extending it; the child takes precedence.
- A class and an interface it implements; the class takes precedence.
- A class and a structure definition with the same fields; the class takes precedence.
- Int and Float; Int takes precedence.
- Anonymous structures where one is a superset of the other; the superset takes precedence.
- Dynamic; any other implementation will take precedence.
- An abstract and its underlying type, or two abstracts with the same underlying type, for traits used in @:generic functions.
- `MyType<T>` and `MyType<SpecificType>`; the specific form takes precedence.

Trait implementations must be imported to be used. Therefore, trait implementations which are not defined either in the same file as (1) the type they're implemented for or (2) the interface they're implementing will not automatically apply without a separate import.

## Impact on existing code

As a new language feature, this would be opt-in and should have no impact on existing code, although changes to the AST may break macros.

## Drawbacks

Resolving to the correct trait implementation can be complex and potentially confusing.

This feature may give users the potential to break external code by implementing external interfaces for external types, which can result in functional changes. In Rust this is dealt with by only allowing implementations where either the type or the trait (or both) was defined within the crate; these would have no possibility of affecting external crates. While Haxe packages are not as isolated as Rust crates, the same strategy may be useful.

## Alternatives

There are other ways of adding functionality to existing types, which are more limited and don't provide the same power as traits:

- Static extension provides utility "methods" to types. These "methods" apply locally via the `using` keyword to a single specific type and are not reflected in the type system. Using statements do not cause the underlying type to unify via structural subtyping, so I can't write a function that takes types having a given method and pass in my `using` type.
- Abstracts can wrap existing types and add new functionality. However, because abstracts cannot implement interfaces, it's impossible to group multiple existing types sharing the same functionality via an interface and abstracts.
- Structural subtyping doesn't allow adding new functionality and won't work with basic types.

This can also be done via wrapper classes that implement interfaces, but at some extra overhead and without the added modularity of traits.

## Opening possibilities

Traits would open the door to more thoroughly utilizing open polymorphism within the Haxe standard library. For example as above, the concept of a `toString()` method, currently a "magic method" with special meaning in `Std.string`, can be encoded as a strongly typed trait in the standard library, with provided implementations for `{toString: Void->String}` and all primitive types. This would preserve the existing behavior while also providing a way to extend it.

Another example is `haxe.io.Bytes`, which currently contains a finite set of methods for adding types to a byte stream; with traits, a native Bytes object could be extended to support any type that implemented `ReadBytes` or `WriteBytes` traits, including non-classes such as structs which would currently require extending Bytes. (This example could be implemented via static extension, but a generic `write` function accepting many different types could not.)

See many other examples of possible future additions in the Rust standard library, including `Drop` (destructors), `From<T>` (type conversion), and `Copy`.

Traits could be even more powerful with the addition of a [polymorphic this type](https://github.com/HaxeFoundation/haxe-evolution/pull/36), allowing the trait to refer to the implementer.

## Unresolved questions

- How would @:autoBuild etc. work on an interface implemented as a trait? This could be dangerous as it allows rewriting external code.
