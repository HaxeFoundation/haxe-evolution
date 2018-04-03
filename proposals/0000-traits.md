# Traits/Type Classes

* Proposal: [HXP-NNNN](NNNN-traits.md)
* Author: [Ben Morris](https://github.com/bendmorris)

## Introduction

A more powerful analog to static extension that enables open polymorphism, modular type specifications and wrapping interfaces and new behavior over existing types.

## Motivation

Traits, or type classes, are a powerful extension of the concept of interfaces from OOP, with some key differences:

- Implementing traits is external to the definition of the type; traits may be implemented for existing types, including basic types, outside of the original type declaration.

Haxe provides several analogous features for extending the functionality of existing types, including static extension and abstracts. However, none of these features is as powerful as traits; specifically, they lack the ability to define a single type which wraps multiple existing types. Adopting the Rust trait system for Haxe will open the door to new ways of defining behavior in a modular and extensible way.

Benefits of traits include:

- Providing an elegant way to express polymorphism over an open set of types, including types which are not classes such as primitives, abstracts or structures.
- Breaking up large class/abstract definitions into more concise blocks by functionality.
- Providing a standard way to extend language-level functionality (see "opening possibilities".)

Note: some OOP languages such as PHP include a "trait" concept, also known as a "mixin", focused around code reuse. This is an orthogonal concept that unfortunately shares the same name. This proposal deals with the specific implementation of traits from [Rust](https://rustbyexample.com/trait.html) as a method of open polymorphism.

## Detailed design

### Declaring traits

From an OOP perspective, traits can easily be understood as analogous to interfaces. Therefore, the simplest way to introduce them to Haxe is to extend the existing interface concept:

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

Given the following interface, the following two implementations are equivalent:

```haxe
interface MyInterface {}

// interface implementation
class MyClass1 implements MyInterface {}

// trait implementation
class MyClass2 {}
implement MyInterface for MyClass2 {}
```

They differ in how the caller is treated; see "implementation" below.

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

### Implementation

An `implement X for Y` block creates a "trait declaration" of interface X for type Y. Trait implementations can be compiled in two ways:

#### Generic trait use

When used as a generic type parameter, a separate version of the function is generated per used implementer; in these functions the trait implementation may be inlined. For example this generic function:

```haxe
@:generic public static function serialize<T:Serializable>(value:T):String
{
	return value.toString();
}
```

When the observed type of parameter T is a trait declaration unifying with the interface (and not an object that implements the interface), the trait declaration's method is inlined, yielding:

```haxe
public static function serialize_Int(value:Int):String
{
	return Std.int(value);
}

public static function serialize_name_String(value:{name:String}):String
{
	return value.name;
}
```

This provides a zero-runtime-cost version of traits at the expense of larger code size.

#### Trait objects

Otherwise, when a value is coerced to an interface (and the value is not an object that implements the interface) a valid trait declaration will be selected, and the value will be coerced to a wrapper object of a class which implements the interface as per the trait declaration.

Given this function,

```haxe
public function serialize(value:Serializable):String
{
	return value.toString()
}
```

The following:

```haxe
class Main
{
	static function main()
	{
		serialize(42);
	}
}
```

would implicitly yield something like:

```haxe
class Serializable_Int implements Serializable
{
	public var val:Int;

	public function new(val:Int)
	{
		this.val = val;
	}

	public function toString()
	{
		return Std.string(this.val);
	}
}

class Main
{
	static function main()
	{
		serialize(new Serializable_Int(42));
	}
}
```

While this example shows an allocation of a wrapper object, in practice these wrapper objects may be cached and reused internally. The `@:generic` version is also available as an alternative to avoid allocation.

### Trait implementation selection and conflict resolution

Both the `@:generic` and wrapped form of using traits require selecting an appropriate trait implementation at compile time. Eligible trait implementations must not conflict; conflicts will be treated as a compile-time error and must be manually disambiguated. Multiple trait implementations are *not* in conflict if they are unambiguous, or if one is strictly "more specific" than the other; i.e., if type A is valid as type B but type B is not valid as type A, type A's implementation will take precedence.

Trait implementations must be imported to be used. Therefore, trait implementations which are not defined either in the same file as (1) the type they're implemented for or (2) the interface they're implementing will not automatically apply without a separate import.

## Impact on existing code

As a new language feature, this would be opt-in and should have no impact on existing code, although changes to the AST may break macros.

## Drawbacks

Resolving to the correct trait implementation can be complex and potentially confusing.

## Alternatives

There are other ways of adding functionality to existing types, which are more limited and don't provide the same power as traits:

- Static extension provides utility "methods" to types. These "methods" apply locally via the `using` keyword to a single specific type and are not reflected in the type system. Using statements do not cause the underlying type to unify via structural subtyping, so I can't write a function that takes types having a given method and pass in my `using` type.
- Abstracts can wrap existing types and add new functionality. However, because abstracts cannot implement interfaces, it's impossible to group multiple existing types sharing the same functionality via an interface and abstracts.
- Structural subtyping doesn't allow adding new functionality and won't work with basic types.

This can also be done via wrapper classes that implement interfaces, but at some extra overhead and without the added modularity of traits.

## Opening possibilities

Traits would open the door to more thoroughly utilizing open polymorphism within the Haxe standard library. For example as above, the concept of a `toString()` method, currently a "magic method" with special meaning in `Std.string`, can be encoded as a strongly typed trait in the standard library, with provided implementations for `{toString: Void->String}` and all primitive types. This would preserve the existing behavior while also providing a way to extend it.

Another example is `haxe.io.Bytes`, which currently contains a finite set of methods for adding types to a byte stream; with traits, a native Bytes object could be extended to support any type that implemented `ReadBytes` or `WriteBytes` traits, including non-classes such as structs which would currently require extending Bytes. (This example could be implemented via static extension, but a generic `write` function accepting many different types could not.)

See many other examples of possible future additions in the Rust standard library, including `Drop` (destructors), `To<T>` and `From<T>` (type conversion), and `Copy`.

Traits could be even more powerful with the addition of a [polymorphic this type](https://github.com/HaxeFoundation/haxe-evolution/pull/36), allowing the trait to refer to the implementer.

## Unresolved questions

None?
