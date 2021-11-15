# Null-safe navigation operator

* Proposal: [HXP-0017](0017-null-safe-navigation-operator.md)
* Author: [Robert Borghese](https://github.com/RobertBorghese)
* Status: [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10479)

## Introduction

Provide [safe-navigation](https://en.wikipedia.org/wiki/Safe_navigation_operator) (aka. optional chaining) to Haxe. This allows for values that may be `null` to be safely accessed, array-accessed, and called without explicitly checking if the value is `null` ahead of time.

```haxe
final iAmNull: Null<String> = null;
final iAmString: Null<String> = "Hello!";

iAmNull?.length; // null
iAmString?.length; // 6

// convoluted method of checking first character
if(iAmNull?.toUpperCase()?.split("")?.[0] != null) {
	trace("iAmNull is not null and len > 0");
}

final imAlsoNull: Null<(Int) -> Void> = null;
imAlsoNull?.(123); // won't be called
```

## Motivation

This feature radically simplifies a common programming pattern with the use of a single operator. Specifically, this pattern is the checking of nullable values prior to accessing or calling; however, this often results in tedious `if` statements:
```haxe
// calling function on nullable variable
if(myObject != null) {
	myObject.runMethod();
}

// accessing member in chain of possibly null values
final info = if(myObject != null && myObject.data != null && myObject.data.result != null) {
	myObject.data.result.confirm();
} else {
	null;
}

// alternatively...
final info = myObject != null ? (myObject.data != null ? (myObject.data.result != null ? myObject.data.result.confirm() : null) : null) : null;
```
The safe navigation operator drastically improves the size and readability of code in these situations. It turns what could be multiple lines or an over-extended line into a smaller expression.
```haxe
// calling function on nullable variable
myObject?.runMethod();

// accessing member in chain of possibly null values
final info = myObject?.data?.result?.confirm();
```
Moreover, it works well with `@:nullSafety(Strict)`, encouraging syntax that checks the value at the moment it's used, preventing issues stemming from modification after the `if` check.
```haxe
@:nullSafety(Strict) {
	var myArray: Null<Array<Int>> = [1, 2, 3];
	if(myArray != null) {
		myArray.push(4); // valid
		if(Math.random() < 0.5) {
			myArray = null;
		}
		myArray?.pop(); // valid
	}
}
```

---
While the feature itself does not provide new functionality, it provides a **single**, concise syntax for multi-nullable chains (as opposed to `if/else`, `?:`, `?(?:):`, etc). As such, I believe it fits perfectly with Haxe's design philosophy.

Furthermore, it's a very popular feature. Currently, out of the seven source-to-source targets Haxe supports, nearly half support safe navigation (JavaScript, C#, and PHP). Of course, whether or not a source target supports this syntax ~~probably~~ doesn't matter when it comes to implementation, but it shows how Haxe is falling behind with some languages it should be an enticing alternative for. This especially applies to JavaScript, one of the most popular targets for Haxe. 

In addition, this doesn't even take into account the more "modern" languages Haxe competes with, the large majority of which support this feature as well: Kotlin, Swift, Ruby, Crystal, Scala, TypeScript, CoffeeScript, Groovy, Dart.

Finally, it should also be noted the feature is popular within the Haxe community. With the rise of [ReallyUniqueName's Safety](https://github.com/RealyUniqueName/Safety) and official incorporation into Haxe with `@:nullSafety`, I think the sooner it's added the better. Especially while things like [null-safe std](https://github.com/HaxeFoundation/haxe/pull/10081) are in the works, and the [null-coalescing operator](https://github.com/HaxeFoundation/haxe-evolution/pull/85) is being considered.

## Detailed design

As demonstrated above, `?.` will act as an alternative to `.` for nullable types. The resulting type will always be `Null<T>`.

Adding this feature will require modification of the `ExprDef` enum to include the safe alternatives to `EField`, `ECall`, and `EArray` through either additional enum values or fields in the enum values.

In terms of specific syntax, there should never be space between the two characters of the operator.
```haxe
final nullString: Null<String> = null;
nullString?.length; // safely returns null (Null<Int>)
nullString? .length; // error
nullString  ?.length; // ok (based on current . behavior)
```

`?.[]` and `?.()` can also be used for array-access or function calls on nullable values as well. The full `?.` is used to prevent conflict with ternary conditions: `a?(b):c` or `a?[b]:c`.

There can be white space between the `?.` and `[` or `(`, but like before, the `?.` operator should remain intact.
```haxe
final nullArray: Null<Array<Int>> = null;
final nullFunc: Null<() -> Void> = null;

nullArray?.[2]; // safely returns null
nullFunc?.(); // safely returns null without calling

nullArray ?. [2]; // valid
nullArray ? . [2]; // invalid
```

If `@:nullSafety` is enabled, the operator should throw an error (or at least a warning) on all types that are not `Null<T>`.
```haxe
@:nullSafety(Strict) {
	final myString: String = "Test";
	myString?.length; // error: "myString" can never be null
}
```

On the other hand, if `@:nullSafety` is enabled, the safe navigation operator will function on `Null<T>` without explicit checks. 
```haxe
@:nullSafety(Strict) {
	final nullArray: Null<Array<Int>> = null;
	nullArray?.length; // valid
	nullArray?.[0]; // valid
}
```

## Impact on existing code

As mentioned in Detailed Design, this feature would require modification of AST; therefore, it may break compatibility with some macros using switch against all `ExprDef` cases.

However, beyond that, adding this feature doesn't invalidate any existing syntax, so there should be no problems.

## Drawbacks

The parsing of the `?.` operator may conflict with float literals (`cond?.1:.2`), so additional logic may need to be incorporated into the parsing of the operator.

With how prevalent null-checks are, it might be a little tedious for developers who choose to optionally "update" their codebase with the new feature. This would not be required and, if anything, would help those trying to make their project fully compatible with `@:nullSafety`.

## Alternatives

Macros can replicate the "safe access" (`?.`) functionality. This is achieved in [Safety](https://github.com/RealyUniqueName/Safety/). Unfortunately, on top of the fact that it can't be used with other macros and affects compiling performance, macros cannot replicate the feature using the standard `?.` syntax and relies upon non-standard syntax like `!.`.

In addition, macro-created safe array-access and function calls require even weirder or cluckier syntax that comes no where near the desired cleanliness of the proposed syntax.

## Unresolved questions

The syntax will be the biggest question for this feature. The use of `?.` for access is pretty consistent across almost all languages with the feature (JavaScript, C#, Python ([proposal](https://www.python.org/dev/peps/pep-0505/)), Kotlin, Swift, TypeScript, CoffeeScript, Groovy, Dart).

However, things begin to diverge when it comes to the safe array-access and function call. JavaScript is the only language that uses `jsFunction?.()` or `jsArray?.[0]`. Since Haxe follows the same syntax, it makes sense to follow its lead, but it is one of the more obscure styles compared to other programming languages. An alternative option would be to provide function-alternatives to all std classes' array-access and function calls (`arr?.get(0)` or `arr?.set(0, 1)`). This would be a similar approach to C# and Kotlin's `myFunction?.Invoke(arg1, arg2)`.

The format of the changes to `ExprDef` are also yet to be determined. 
