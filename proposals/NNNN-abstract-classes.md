# Abstract Classes

* Proposal: [HXP-NNNN](NNNN-abstract-classes.md)
* Author: [Aleksandr Kuzmenko](https://github.com/RealyUniqueName)

## Introduction

Introduce classes that provide incomplete implementation and cannot be instantiated directly.
To instantiate an abstract class one has to extend it with supplementing implementation first.

## Motivation

Requests for this feature keep coming up from time to time.
And while Haxe doesn't have such a feature even standard library uses this concept.

E.g. here is the doc for `haxe.io.Input`:
> An Input is an abstract reader. See other classes in the `haxe.io` package for several possible implementations.

It could lead to inconsistencies like some abstract methods throw "not implemented" exceptions:
```haxe
/**
	Read and return one byte.
**/
public function readByte():Int {
	#if cpp
	throw "Not implemented";
	#else
	return throw "Not implemented";
	#end
}
```
while others just do nothing:
```haxe
/**
	Close the input source.

	Behaviour while reading after calling this method is unspecified.
**/
public function close():Void {}
```
And it's not obvious if an empty `close` method is a part of a valid implementation or if it's an abstract method, which must be overridden.

As a consequence I can't tell if [Stdout implementation](https://github.com/HaxeFoundation/haxe/blob/b832af9/std/neko/_std/sys/io/Process.hx#L56)
for Neko is missing `close` implementation by mistake. Because `Stdin` implementation in that same file has `close` implemented.

With an upcoming [asys feature](https://github.com/HaxeFoundation/haxe/pull/8832) standard library is likely to get a
[few](https://github.com/HaxeFoundation/haxe/blob/b832af9/std/haxe/io/Duplex.hx#L12)
[more](https://github.com/HaxeFoundation/haxe/blob/b832af9/std/haxe/io/Readable.hx#L11)
[abstract](https://github.com/HaxeFoundation/haxe/blob/b832af9/std/haxe/io/Writable.hx#L10)
[classes](https://github.com/HaxeFoundation/haxe/blob/b832af9/std/haxe/io/Transform.hx#L26).

Five of Haxe targets have native abstract classes. Writing correct externs for such classes doesn't seem viable without macros.

Haxe developers use `throw "Not implemented"` and similar constructs to emulate abstract classes [quite a lot](https://github.com/search?l=&p=2&q=not+implemented+language%3AHaxe&type=Code). That means some of these runtime errors survive to production.

With a proper abstract classes all of that could be fixed and unified.

## Detailed design

Class becomes an abstract class if it has `abstract` access modifier:
```haxe
abstract class SomeAbstr {
```
It does not matter if a class has `private` modifier before or after `abstract`:
```haxe
private abstract class SomeAbstr {
abstract private class AnotherAbstr {
```
Abstract method is a method that is declared without an implementation and has `abstract` access modifier:
```haxe
abstract function implementMe():Void;
```
Abstract method is not allowed to have an implementation
```haxe
abstract function implementMe2():Void { // Error: abstract method cannot have an implementation
	trace('hello');
}
```
`abstract` access modifier position among other access modifiers of a method does not matter:
```haxe
abstract public function implementMe1():Void; //ok
public abstract function implementMe2():Void; //also ok
```
Static methods cannot be abstract:
```haxe
abstract class SomeAbstr {
	static abstract function implementMe():Void; // Error: static functions cannot be abstract
}
```
Abstract classes may contain static fields just like normal classes:
```haxe
abstract class SomeAbstr {
	static public var field:String = 'hello';
	static public function method():Void {}
}
```
Abstract classes may or may not contain abstract methods.
```haxe
abstract class SomeAbstr {
	abstract function implementMe():Void;
}
abstract class AnotherAbstr { } // also ok
```
Abstract classes cannot be instantiated:
```haxe
abstract class Main {
	static function main() {
		var m = new Main(); // Error: cannot instantiate abstract class Main
	}

	public function new() {}
}
```
Abstract classes may contain an implementation, but still cannot be instantiated:
```haxe
abstract class Greeter {
	public final name:String;
	public function new(name:String) {
		this.name = name;
	}
	public function greet() trace('Hello, $name!');
}

class Main {
	static public function main() {
		new Greeter('world'); //Error: cannot instantiate abstract class Greeter
	}
}
```
If a class contains abstract methods, then the class itself must be declared `abstract`.
```haxe
class Some {
	abstract function implementMe():Void; // Error: non-abstract class Some cannot contain abstract functions
}
abstract class Some {
	abstract function implementMe():Void; // ok
}
```
If a class extends an abstract class, but does not implement all abstract methods, then the class itself must be declared abstract:
```haxe
abstract class Abstr {
	abstract function implementMe():Void;
}
class Concrete extends Abstr {} // Error : Concrete is missing an implementation for function implementMe
abstract class AnotherAbstr extends Abstr {} // ok
```
Abstract class is not required to implement all the methods of interfaces that class implements.
Instead missing interface implementations are implicitly treated as abstract methods:
```haxe
interface IFace {
	function implementMe():Void;
}
abstract class Abstr implements IFace {} // ok
class Concrete extends Abstr {} // Error : Concrete is missing an implementation for function implementMe
```
`abstract` access modifier cannot be used with `final` or `inline` access modifiers:
```haxe
final abstract class Abstr { // Error: abstract class cannot be final
	final abstract function implementMe():Void; // Error: abstract function cannot be final
	inline abstract function doSomething():Void {} // Error: inline function cannot be abstract
	inline abstract function doSomething():Void; // Error: abstract function cannot be inlined
}
```
It's not allowed to inline abstract functions at call site:
```haxe
abstract class Abstr {
	abstract function implementMe():Void;
}
//<...>
var a:Abstr = getAbstr();
inline a.implementMe(); // Error: abstract function cannot be inlined
```
Interfaces and their fields cannot have `abstract` access modifier:
```haxe
abstract interface IFace { // Error: interfaces don't need abstract modifier
	abstract function implementMe():Void; // Error: interface fields don't need abstract modifier
}
```
If a class provides an implementation for an abstract method it does not need `override` keyword as no actual implementation is overridden.
```haxe
abstract class Abstr {
	abstract function implementMe():Void;
}
class Concrete extends Abstr {
	function implementMe():Void { //no "override"
		trace('implemented');
	}
}
```
Abstract classes may define abstract getters and/or setters:
```haxe
abstract class Abstr {
	public var prop1(get,set):Int;
	function get_prop1() return Std.random(10);
	abstract function set_prop1(value:Int):Int;

	public var prop2(get,set):Int;
	abstract function get_prop2():Int;
	abstract function set_prop2(value:Int):Int;
}
```
Abstract classes don't imply any special rules for constructors.

## AST changes

1. Add `isAbstract` flag to type definitions:
   * As `?isAbstract:Null<Bool>` argument to `TDClass` constructor of [`haxe.macro.TypeDefKind`](https://github.com/HaxeFoundation/haxe/blob/518d019/std/haxe/macro/Expr.hx#L957) (breaking change for macros)
   * OR as `var ?isAbstract:Null<Bool>` field to [`haxe.macro.TypeDefinition`](https://github.com/HaxeFoundation/haxe/blob/518d019/std/haxe/macro/Expr.hx#L892) since it already has optional `isExtern`.
2. Add `AAbstract` constructor to [`haxe.macro.Access` enum](https://github.com/HaxeFoundation/haxe/blob/518d019/std/haxe/macro/Expr.hx#L813)
3. Add `var isAbstract:Bool` to [`haxe.macro.BaseType`](https://github.com/HaxeFoundation/haxe/blob/518d019/std/haxe/macro/Type.hx#L351)
4. Add `var isAbstract:Bool` to [`haxe.macro.ClassField`](https://github.com/HaxeFoundation/haxe/blob/518d019/std/haxe/macro/Type.hx#L188)
5. Make corresponding changes on the ocaml side.

## Impact on existing code

Proper implementation as described in [AST changes](#AST-changes) would introduce breaking changes for macro development.
Although, proposed changes seem to be minor and should not affect most of Haxe projects.

Alternatively we can start with a meta `@:abstract` instead of a proper access modifier.
Or parse an access modifier into a meta to avoid any breaking changes for macros while still having a proper access modifier in Haxe code.

## Drawbacks

Using `abstract` keyword for this feature may be confusing especially in a documentation since this keyword is already used for [Abstract types](https://haxe.org/manual/types-abstract.html)

But using any other keyword would be even more confusing since "abstract classes" is a common term for almost any OOP programming language.

This probably can be solved to some extent by _always_ using "abstract" accompanied with the correct word (either "abstract type" or "abstract class")
in docs or speeches to form a strong distinction between these two features in the "community mind".

## Alternatives

It is possible to implement abstract classes with macros ([most known implementation by @AndyLi](https://gist.github.com/andyli/5011520)).

But a macro implementation has obvious disadvantages:
* Could not be used while developing other macros
* Affects compilation time
* Unwanted in the standard library
* Not supported by the compiler to generate target-native abstract classes for better interoperability with the target platform.