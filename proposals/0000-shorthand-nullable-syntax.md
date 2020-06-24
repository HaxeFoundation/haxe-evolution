# Shorthand nullable-type syntax

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Robert Borghese](https://github.com/RobertBorghese)

## Introduction

Allow `T?` to act as shorthand for `Null<T>`.

```haxe
function GetFirst(arr: Array<Int?>?): Int? {
	if(arr != null) {
		for(i in 0...arr.length) {
			var v: Int? = arr[i];
			if(v != null) return v;
		}
	}
	return null;
}
```

## Motivation

There are two main motivations behind this proposal. Firstly, type parameters are tedious to write out, and they can become overwhelming long and frustrating. While this is natural and usually necessary for accurately describing types with specific type parameters, I think an argument can be made to cut down on these by providing a standard shorthand for the most common generic-type holder: `Null<T>`.

An example would be the function argument's type shown in the introduction. In Haxe's current state, the argument's type needs to be written out like this: `Null<Array<Null<Int>>>`. While this may seem like a niche case, dealing with types like this are quite common when dealing with JSON (even more annoying when an entire anonymous structure type is written out), or null-safe Haxe code in general. For example, `Null<T>` must be explicitly assigned to variables in many cases where type-inference cannot be relied upon (such as receiving information from lower scopes like below).

```haxe
var Data: Null<Int> = null;
for(i in 0...MyArray.length) {
	if(MyArray[i] == 23) {
		Data = i;
		break;
	}
}
if(Data != null) {
	// do something
}
```

Long story short, due to the addition of null-safety in Haxe, there is an increased prevalence of `Null<T>` within Haxe code, and it would be nice to reduce code bloat and not have to write unnecessarily larger type descriptions.

---

The second motivation is that this syntax meshes better with Haxe's design philosophy. From what I understand, Haxe is looking to be expressive, yet simple with how it is portrayed. It should be readable to those who have never programmed in Haxe before. While `Null<T>` follows a more common pattern for describing a type "wrapper", and type parameters are a common pattern in nearly all statically-typed languages, the `T?` pattern does an overwhelmingly better job at describing the intent of the code in a much simpler manner.

Unlike getters/setters or anonymous functions, the syntax for nullable types in other languages with null-safety is pretty much unanimous. Going down the [TIOBE Index](https://www.tiobe.com/tiobe-index/) list for June 2020, every language in the top 50 that has `null` and null-safety uses the `T?` syntax (ex: C#, Swift, Dart, Kotlin, etc.). Everything else either has nullability for all objects (ex: Java) or has `null` removed from the language (ex: Rust).

In other words, `T?` is a well established syntax that correlates to nullability and null-safety. Especially once .Net 5 releases, C# 8+ becomes the new standard, and massive frameworks like [Unity](https://forum.unity.com/threads/unity-c-8-support.663757/#post-4444186) begin to support it, null-safety will become even more of a buzz-phrase and its correlating syntax will be even more recognizable and mainstream.

Now, the point being made is not that Haxe should simply copy everything that's popular; it's that since Haxe already supports null-safety, it would make sense to provide the syntax that's not only more simple and easy to read/write, but also what's essentially become the standard (especially in languages with such similar user-bases like Kotlin and Dart).

## Detailed design

Internally, `T?` would be identical to `Null<T>`. I would imagine once it is parsed, its AST would be synonymous with `Null<T>`.

In terms of its syntax design, it would simply be a `?` at the end of any Type:
```haxe
class MyClass {
	static var list: Array<MyClass>?;
	static var number: Int?;

	function new() {
		if(list == null) {
			list = [];
		}
		list.push(this);

		number = DoThing(number, 10);
	}

	function DoThing(a: Float?, b: Float): Float? {
		var result: Float?;
		if(a != null) {
			result = b;
		}
		return result;
	}
}
```

### Whitespace

Similar to other languages, there can be whitespace between the type and the `?`; however, no whitespace should be the standard:
```haxe
// all valid
var a: Int? = null;
var b: Int ? = null;
var c: Int   
    

    ? = null;
```

### Type Parameter Placement

When used with Types with type arguments, the `?` should be after the `<...>`.
```haxe
var a: Map<String, Int>? = null; // valid (Null<Map<String, Int>>)
var b: Map?<String, Int> = null; // invalid
var c: Map<String, Int?>? = null; // valid (Null<Map<String, Null<Int>>>)
```

### Nullable Redundancy

In the case of multiple `?` on the exact same Type (`Int???`), there are multiple solutions. It could throw an error (like C#), it could give a warning and default to the behavior of a single `?` (like Kotlin), or it could actually stack the `Null<T>` wrapper class like this: `Null<Null<Null<Int>>>` (like Swift). Based on the reactions of C# and Kotlin, it seems pretty safe to say there is no feasible reason to use multiple `Null<T>` wrappers, and in most cases this is probably user error.

With that being said, Haxe already appears to handle `Null<Null<T>>` the same as `Null<T>` so perhaps it's fine that way? If leaving it that way is all that takes to make this feature to be included, that's perfectly fine, but perhaps throwing an error should be considered, especially to help fix the over-redundancy some users may implement:
```haxe
var a: Int? = null;
var a: Null<Int> = null;
var b: Int?? = null; // invalid
var c: Null<Int>? = null; // invalid
var d: Null<Int?> = null; // invalid
```

### Potential Ternary Condition Conflict Resolution

The only hypothetical case there could be for operator confusion/conflict would be when a Type is used as a value (as an input for `Class<T>`) and its `?` may be confused with the ternary conditional: `A ? B : C`. Since `A` MUST be a `Bool`, I can't imagine there being a valid scenario where there would be `Type? ? B : C`, so it's probably safe to assume that every sequential `?` after a Type correlates to it.

If something along the lines in the code shown below is a valid conflict, then `T?` should be disabled when using a Type as a value. Instead, users should be forced to use `Null<T>`. These are very rare occurrences, so it should not affect the overall convenience this feature would provide.
```haxe
// Is this even valid Haxe code?
var a: Class<T> = 32 == 32 ? Int? : Float?;

// Maybe something like this could cause issues?
// Once again, not even sure if valid.
var b: Class<T> = Int?;
var c: Class<T> = b == Int? ? Int? : Float?;
var d: Class<T> = b == Int ? Int? : Float?;

// Maybe there would be some issue with a cast?
// Casting with a Type uses parentheses though...
var e: Int = someInt == cast(someOtherInt, Int?) ? 1 : 2;
```

## Impact on existing code

Since this would not replace or remove `Null<T>` (only work as an alternative), all existing code should still function like normal.

## Drawbacks

Since `T?` converts to `Null<T>`, I cannot imagine any new internal issues that may occur. All existing null-safety progress should work the same. The only drawbacks may be having to rewrite existing documentation, some users being a little confused and overly-redundant with the feature (`Null<T>?`), or some conflict with the aforementioned ternary condition operator.

## Alternatives

Based on the previously described motivation, `T?` appears to be the most superior option in terms of a syntax change, but perhaps the implementation could be different? Instead of adding this to the compiler directly, macros could be expanded to allow such a feature to be implemented on the user's end? However, in my opinion, since `Null<T>` is already a top-level feature, it shouldn't be too much of a leap to add `T?` to the language.

## Opening possibilities

Since this would involve creating a system to read postfix operators to Types, I suppose this could open possibilities to expand macro features.

It could also lead to the development of a feature that allows for `T[]` to be shorthand for `Array<T>`, which I would personally love to see, but don't expect to happen.

## Unresolved questions

The only unresolved part would be part 4 of "Detailed design" when choosing how to handle `Null<T>` redundancy. Though, as mentioned, it could just be left alone and there should not be any problems. 
