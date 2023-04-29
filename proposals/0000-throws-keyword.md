# ::The `throws` keyword::

* Proposal: [HXP-0000](0000-throws-keyword.md)
* Author: [atpx8](https://github.com/SomeoneSom)

## Introduction

Introduce typed and explicit error throwing via the `throws` keyword.

Based off of [this TypeScript suggestion.](https://github.com/microsoft/TypeScript/issues/13219)

## Motivation

Currently, when writing a function that can error, or dealing with functions
that can error, theres no easy way to signify or check that a function can error
and the type of the error it can throw.

While this effect can be achieved through doc comments, most libraries are not
extensively documented, and those that are are not documented in a standardized
matter, potentially leading to confusion. Comments also dont force the developer
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
As you can see, its not obvious that this function can error from a first glance.
This can lead to unexpected panics down the line when an unsuspecting programmer
accidentally passes an empty array to this function.

Option 2. Doc comments
```haxe
/**
    This function finds the maximum element of an array.

    @param arr The array in question.
    @return The maximum element of the array.
    @throws A `String` if the array is empty.
**/
public static function maxOfArray<T>(arr:Array<T>):T {
    if (arr.length == 0) {
        throw "Array must have an element in it!";
    }
    // ...
}
```
While this is definitely better, and a valid solution, its also still not great.
For one, while it would be great for every library to have nice and detailed doc
comments, that simply isnt the case for most libraries. The bigger downside though
is that a comment is not an actual type check. It could be the case that this function
throws something different and the comment was never updated, or that the function
doesn't throw anything at all.

Option 3. The `throws` keyword
```haxe
public static function maxOfArray<T>(arr:Array<T>):T throws String {
    if (arr.length == 0) {
        throw "Array must have an element in it!";
    }
    // ...
}
```
This is what I am proposing. The main two differences between this approach and a doc
comment is that:
1. The `throws` keyword is much easier to code, therefore having easier adoption.
2. The `throws` keyword is more standard than a doc comment.
3. The `throws` keyword forces developers to stay true to their word via a type check.

The `throws` keyword can also error if there is no reachable `throw` statement in the function
itself, in order to keep the keyword meaningful.

## Detailed design

Describe the proposed design in details the way language user can understand
and compiler developer can implement. Show corner cases, provide usage examples,
describe how this solution is better than current workarounds.

## Impact on existing code

If the `throws` keyword is made mandatory, then it would break any older code that
uses `throw`. However, migration would be easy, as it's essentially just a
return type but for errors.

## Drawbacks

This syntax is somewhat unconventional, and may take some time getting used to.

## Alternatives

An alternative would be implementing something similar to Rust's `Result` enum.
However, since Haxe already has `catch` and `throw`, implementing this would
make it so there are to seperate ways to do the same thing, and it would confuse
a lot of the users of the language, while this takes the existing method
for error throwing and improves on it.
