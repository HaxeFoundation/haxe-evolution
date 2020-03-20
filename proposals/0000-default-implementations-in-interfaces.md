# Default implementations in interfaces

* Proposal: HXP-NNNN
* Author: [shohei909](https://github.com/shohei909)

## Introduction

An interface method can have a body.

```haxe
interface A {
	function a():Void {
		trace("A");
	}
}

class B implements A {}
```

### Background

Some other languages have the feature.

* [Java](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html)
* [Kotlin](https://kotlinlang.org/docs/reference/interfaces.html)
* [C#8.0](https://docs.microsoft.com/en-us/dotnet/csharp/tutorials/default-interface-methods-versions)([proposal](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-8.0/default-interface-methods))

This proposes a way like Java and Kotlin.

## Motivation

Some are common with [abstract class](https://github.com/RealyUniqueName/haxe-evolution/blob/abstract-classes/proposals/NNNN-abstract-classes.md#motivation). It enables a type to have both abstract functions and implementations.

Although C # and Java already have abstract classes, they added default implementations for interfaces later. 
As you can see from that, it has advantages not found in abstract classes.

### Updating interface without breaking change

When you want to add new functions to interfaces in your library, you can update without breaking change.

### Mixin / Multiple inheritance

Implementing two interfaces which have default methods enables multiple inheritance.

Actually, haxe already has mixin feature `@:using` for interface. However, it is confusing when function names conflict:

```haxe
@:using(Main.UTools)
interface U {}
class UTools {
    public static function sample(u:U):Void {
        trace("U");
    }
}
@:using(Main.VTools)
interface V {}
class VTools {
    public static function sample(v:V):Void {
        trace("V");
    }
}

class UV implements U implements V {
    public function new() {}
}
class VU implements V implements U {
    public function new() {}
}

class VUV extends VU implements V {}

class Main {
    static function main() {
        new UV().sample();  // V
        new VU().sample();  // U
        new VUV().sample(); // U
    }
}
```

`@:using` for interface has

* no `override` and no subtyping polymorphism
* no conflict error

Default implementations in interfaces avoid these problems.

## Detailed design

### Output

Compiler converts first example to:

```haxe
interface A {
	function a():Void;
}

class _A_Impl {
	public static function a(self:A):Void {
		trace("foo");
	}
}

class B implements A {
	public function a():Void {
		_A_Impl.a(this);
	}
}
```

### Override

Overriding function requires `override` keyword.

```haxe
interface A {
	function a():Void {
		trace("foo");
	}
}

class B implements A {
	public override function a():Void {
		trace("bar");
	}
}

interface C extends A {
	override function a():Void {
		trace("baz");
	}
}
```

### Static call

Interface functions can be called as static function with an additional first argument `this`.

```haxe
interface A {
	function a():Void {
		trace("foo");
	}
}

class X {
	public static function x(a:A):Void {
		A.a(a); // foo
	}
}
```

### Conflict

To resolve conflicts requires overriding.

```haxe
interface Y1 {
	function y():Void {
		trace("foo1");
	}
}

interface Y2 {
	function y():Void {
		trace("foo2");
	}
}

class Z implements Y1 implements Y2 {} // Compile Error
```

The following is OK:

```haxe
class Z implements Y1 implements Y2 {
	public override function y() {
		Y1.y(this); // foo1
	}
}
```

## Impact on existing code

None.

## Alternatives

* [Abstract class](https://github.com/HaxeFoundation/haxe-evolution/pull/69)
* `@:using` metadata for interface
* Macro

## Drawbacks

* No support target native abstract class
* Existing starndard library change will be breaking, because of changing `extends` to `implements`.
