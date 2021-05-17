
# Safe Navigation Operator

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Robert Borghese](https://github.com/RobertBorghese)

## Introduction

Provide [safe-navigation](https://en.wikipedia.org/wiki/Safe_navigation_operator) (aka. optional chaining) to Haxe. This allows for values that may be `null` to be safely accessed, array-accessed, and called without explicitly checking if the value is `null` ahead of time.

```haxe
final iAmNull: Null<String> = null;
final iAmString: Null<String> = "Hello!";

iAmNull?.length; // null
iAmString?.length; // 6

// convoluted method of checking first character
if(iAmNull?.toUpperCase()?.split("")?[0] != null) {
	trace("iAmNull is not null and len > 0");
}

final imAlsoNull: Null<(Int) -> Void> = null;
imAlsoNull?(123); // won't be called
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
The safe navigation operator drastically improves the size and readability of code in these situations. It turns what could be multiple lines or an over-extended line into a smaller expression. Moreover, it works well with `@:nullSafety(Strict)`, encouraging syntax that checks the value at the moment it's used, preventing issues stemming from modification after the `if` check.
```haxe
// calling function on nullable variable
myObject?.runMethod();

// accessing member in chain of possibly null values
final info = myObject?.data?.result?.confirm();
```
---
While the feature itself does not provide new functionality, it provides a **single**, concise syntax for multi-nullable chains (as opposed to `if/else`, `?:`, `?(?:):`, etc). As such, I believe it fits perfectly with Haxe's design philosophy.

Furthermore, it's a very popular feature. Currently, out of the seven source-to-source targets Haxe supports, nearly half support safe navigation (JavaScript, C#, and PHP). Of course, whether or not a source target supports this syntax ~~probably~~ doesn't matter when it comes to implementation, but it shows how Haxe is falling behind with some languages it should be an enticing alternative for. This especially applies to JavaScript, one of the most popular targets for Haxe. 

In addition, this doesn't even take into account the more "modern" languages Haxe competes with, the large majority of which support this feature as well: Kotlin, Swift, Ruby, Crystal, Scala, TypeScript, CoffeeScript, Groovy, Dart.

Finally, it should also be noted the feature is popular within the Haxe community. With the rise of [ReallyUniqueName's Safety](https://github.com/RealyUniqueName/Safety) and official incorporation into Haxe with `@:nullSafety`, I think the sooner it's added the better. Especially while things like [null-safe std](https://github.com/HaxeFoundation/haxe/pull/10081) are in the works, and the [null-coalescing operator](https://github.com/HaxeFoundation/haxe-evolution/pull/85) is being considered.

## Detailed design

As demonstrated above, `?.` will act as an alternative to `.` for nullable types. There should never be space between the two characters of the operator. The resulting type will always be `Null<T>`.
```haxe
final nullString: Null<String> = null;
nullString?.length; // safely returns null (Null<Int>)
nullString? .length; // error
nullString  ?.length; // ok (based on current . behavior)
```

`?[]` and `?()` can also be used for array-access or function calls on nullable values as well. There should be no whitespace between the question mark and square bracket (`?[`).
```haxe
final nullArray: Null<Array<Int>> = null;
final nullFunc: Null<() -> Void> = null;

nullArray?[2]; // safely returns null
nullFunc?(); // safely returns null
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
	nullArray?[0]; // valid
}
```

## Impact on existing code

Adding this feature will probably require additional values to be added to the `ExprDef` enum or modification of other AST stuff. So it may break macro code using switch statements over every current `ExprDef` value, etc. 

However, beyond that, adding this feature doesn't invalidate any existing syntax, so there should be no problems.

## Drawbacks

With how prevalent null-checks are, it might be a little tedious for developers who choose to optionally "update" their codebase with the new feature. This would not be required and, if anything, would help those trying to make their project fully compatible with `@:nullSafety`.

## Alternatives

Macros can replicate the "safe access" (`?.`) functionality. This is achieved in [Safety](https://github.com/RealyUniqueName/Safety/). Unfortunately, on top of the fact that it can't be used with other macros and affects compiling performance, macros cannot replicate the feature using the standard `?.` syntax and relies upon non-standard syntax like `!.`.

In addition, macro-created safe array-access and function calls require even weirder or cluckier syntax that comes no where near the desired cleanliness of the proposed syntax.

## Unresolved questions

The syntax will be the biggest question for this feature. The use of `?.` for access is pretty consistent across almost all languages with the feature (JavaScript, C#, Python ([proposal](https://www.python.org/dev/peps/pep-0505/)), Kotlin, Swift, TypeScript, CoffeeScript, Groovy, Dart).

However, things begin to diverge when it comes to the safe array-access and function call. The currently proposed style for safe array-access is used by C#, Groovy, and Swift. While the currently proposed safe function call is used by CoffeeScript, Visual Basic .NET, and Swift (again). Overall, the proposed syntax is the most consistent amongst major programming languages. However, if it's not viable or desired for whatever reason, here are some alternatives based on other languages:

JavaScript keeps the `?.` in all instances of safe navigation. For example: `jsFunction?.()` or `jsArray?.[0]`. While it looks weirder, it's much more consistent and associates the concept with a consistent operator. With how similar Haxe's syntax is to JavaScript's, and based on JavaScript's overwhelming popularity, an argument could be made to use this syntax instead. Though, please note this syntax is not used by any other major programming languages.

Ruby and Crystal use a `&.` operator. This is presumably due to the fact methods and variables can end in `?` in those languages, so it's not something Haxe would need to worry about. It's still an interesting alternative in case `?.` can't work for whatever reason.

When it comes to safe function calls, C# and Kotlin function objects have an `Invoke` member that calls the function. So safe function calls work like: `myFunction?.Invoke(arg1, arg2)`. Haxe does not have an equivalent, so this syntax is not likely viable.

Raku (Perl 6) uses the `.?` operator.

Scala uses the `.?.` operator.
