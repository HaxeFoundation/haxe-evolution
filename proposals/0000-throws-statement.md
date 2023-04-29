# The `throws` keyword

* Proposal: [HXP-0000](0000-throws-keyword.md)
* Author: [atpx8](https://github.com/SomeoneSom)

## Introduction

Introduce typed and explicit error throwing via the `throws` keyword.

Based off of the Java keyword of the same name.

## Motivation

Currently, when writing a function that can error, or dealing with functions
that can error, there is  no easy way to signify or check that a function can error
and the type of the error it can throw.

While this effect can be achieved through doc comments, most libraries are not
extensively documented, and those that are are not documented in a standardized
matter, potentially leading to confusion. Comments also do not force the developer
to stay true to what the function should throw as an error, if it even throws an
error at all.

Let's take this piece of code as an example.

Option 1. "Hidden" catch
```haxe
public static function maxOfArray<T>(arr:Array<T>):T {
    if (arr.length == 0) {
        throw "Array must have an element in it!";
    }
    // ...
}
```
As you can see, it is not obvious that this function can error from a first glance.
This can lead to unexpected panics down the line when an unsuspecting programmer
accidentally passes an empty array to this function.

Option 2. Doc comments
```haxe
/**
    This function finds the maximum element of an array.

    @param arr The array in question.
    @return The maximum element of the array.
    @throws exception If the array is empty.
**/
public static function maxOfArray<T>(arr:Array<T>):T {
    if (arr.length == 0) {
        throw "Array must have an element in it!";
    }
    // ...
}
```
While this is definitely better, and a valid solution, it is also still not the best.
For one, while it would be great for every library to have nice and detailed doc
comments, that simply is not the case for most libraries. The bigger downside though
is that a comment is not an actual type check. It could be the case that this function
throws something different and the comment was never updated, or that the function
does not throw anything at all.

Option 3. The `throws` keyword
```haxe
public static function maxOfArray<T>(arr:Array<T>):T throws haxe.ValueException {
    if (arr.length == 0) {
        throw "Array must have an element in it!";
    }
    // ...
}
```
This is what I am proposing. The `throws` keyword (and by extension, the `throws` statement)
has quite a few advantages over a doc comment.

- Using this new statement comes with much less baggage, as this is essentially just adding
a second return type, versus having to write detailed documentation. This can lead to easier adoption.
- The `throws` statement is more standardized than a doc comment, where different developers
might write their documentation differently, which can lead to confusion.
- The `throws` statement forces a developer to make sure that they are both actually
throwing an error, and the type of the error is what they say it is. A doc comment has no
such safeguards in place.

## Detailed design

The `throws` statement should come after the return type and before the opening bracket
of the function code. For arrow functions, it should come after the arguments and before
the arrow.

Implementing this would require adding another optional field to `haxe.macro.Expr.Function`.

The `throws` statement can be implemented without any type hint. In this case, it defaults
to `haxe.Exception`.
```haxe
function example() throws {/* ... */}
// Is equivalent to
function example() throws haxe.Exception {/* ... */}
```

If the `throws` statement is used with a type that does not derive from `haxe.Exception`,
an error should be thrown, due to Haxe automatically boxing non-exceptions into instances
of `haxe.ValueException`.
```haxe
function example() throws String {/* ... */} // error: functions may only throw types that derive from `haxe.Exception`
```

If the `throws` statement is used when a function does not have a reachable `throw` expression,
an error should be thrown, as `throws` should always indicate that a function can error.
```haxe
function example() throws {return;} // error: functions may only use `throws` when they can throw an error
```

However, if a function can error yet does not use the `throws` statement, only a warning
should be output. This is to ease migration, as making this an error would be a massive
breaking change. This can be changed to an error in later versions of Haxe though.
```haxe
function example() {throw "";} // warning: function can throw an error yet does not use the `throws` statement
```

If a function throws an error that does not unify with the type in the `throws` statement,
a type error should be thrown.
```haxe
function example() throws haxe.ValueException {throw new haxe.Exception("")} // error: haxe.Exception should be haxe.ValueException
```

## Impact on existing code

Since the `throws` keyword is not mandatory for function that error, this should have
no impact on existing code. However, if it eventually becomes mandatory, it would
be a major breaking change.

## Drawbacks

For Haxe coders not familiar with Java, this syntax may take some time getting used to.

## Alternatives

An alternative would be implementing something similar to Rust's `Result` enum.
However, since Haxe already has `catch` and `throw`, implementing this would
make it so there are to seperate ways to do the same thing, and it would confuse
a lot of the users of the language, while this takes the existing method
for error throwing and improves on it.

## Unresolved questions

It is unclear whether this new keyword should allow for multiple error types, like it does
in Java. However, this might require an overhaul of the type system, as to my knowledge, it
is currently not equipped to handle a possibility of one of multiple types at the moment.
