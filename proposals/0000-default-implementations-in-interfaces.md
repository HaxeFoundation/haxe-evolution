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

## Motivation

Same as [abstract class](https://github.com/RealyUniqueName/haxe-evolution/blob/abstract-classes/proposals/NNNN-abstract-classes.md#motivation).

### Advantages
* Multiple inheritance
* Minimal AST change
* No breaking IDE
* No confusing keyward

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

Interface functions can be called as static function.

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
* Macro

## Drawbacks

None.