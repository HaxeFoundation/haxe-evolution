# Static Extension Meta Functions

* Proposal: [HXP-NNNN](NNNN-static-extension-meta-functions.md)
* Author: [lublak](https://github.com/lublak)

## Introduction

Allow the possibility to use the meta function,
based on the abstraction, also in static extension classes.

## Motivation

Currently, operator overloading, implicit casts
and array access are reserved for abstractions only.
However, it would also be very practical to provide this possibility via static extension classes,
which can then be consulted with using.

Just one example of how the respective functions could look:

```haxe
class UsingWithMetaFunction {
	@:to public static inline function to(str:String):Int {
		return Std.parseInt(str);
	}

	@:from public static function from(i:Int):String {
		return Std.string(i);
	}

	@:op(A-B) public static inline function sub(str:String, i:Int) {
		return str.substr(0, -i);
	}

	@:arrayAccess
	public static inline function get(str:String, key:Int):Int {
		return str.charCodeAt(key);
	}
	@:arrayAccess
	public static inline function set(str:String, k:Int, v:Int):String {
		return str.substring(0, k) + String.fromCharCode(v) + str.substring(k+1);
	}
}
```

## Detailed design

Basically, the implementation could work via a kind of lookup.
If an operator, an implicister cast to a type or an array acces occurs, it checks if the functions that have been included by using contain the respective meta.
If so, the respective function should be called.

## Impact on existing code

Basically, nothing should break.
But should someone use the meta data in these classes for other information problems could occur here.

## Drawbacks

None.

## Alternatives

Basically, the class can be wrapped with an abstract.
But then you lose the functionalities of a class. (As an example: extends)

## Unresolved questions

None.
