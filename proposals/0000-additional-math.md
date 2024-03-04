# Additional Mathematical Constants and Functions

* Proposal: [HXP-0000](0000-additional-math.md)
* Author: [EliteMasterEric](https://github.com/EliteMasterEric)

## Introduction

The contents of the standard library's `Math` module are minimal, and there are several functions which should be added to it. There are also modifications which could be made to existing functions to improve convenience.

## Motivation

The standard library's `Math` module provides a standard, tested, performant implementation of each of its mathematical functions on all the supported platforms, however there are many functions which other platforms include in their equivalents to the `Math` module which Haxe does not. There are also several functions which are inconvenient to use in a type-safe context, due to how Math mixes the use of integers and floating-point numbers. Expanding `Math` with the new functions provided below, and backwards-compatible modifications to existing functions, would improve convenience for users.

## Detailed design

The following additional mathematical constants:

```haxe
/**
 * Euler's constant.
 */
static var E(default, null):Float;

/**
 * Tau, equivalent to `2 * PI`.
 */
static var TAU(default, null):Float;

/**
 * The natural log of 10.
 */
static var LN10(default, null):Float;

/**
 * The natural log of 2.
 */
static var LN2(default, null):Float;

/**
 * The logarithm base 10 of Euler's constant.
 */
static var LOG10E(default, null):Float;

/**
 * The logarithm base 2 of Euler's constant.
 */
static var LOG2E(default, null):Float;

/**
 * The square root of 1/2.
 */
static var SQRT1_2(default, null):Float;

/**
 * The square root of 2.
 */
static var SQRT2(default, null):Float;
```

The following additional floating-point mathematical functions:

```haxe
/**
 * Returns the cube root of a number.
 */
public static function cbrt(x:Float):Float;

/**
 * Computes the square root of the sum of the squares of its arguments.
 * NOTE: This utilizes haxe.Rest to accept any number of arguments.
 */
public static function hypot(...args:Float):Float;

/**
 * Linearly interpolate between two values.
 * @param t The interpolation factor.
 * @param a The starting value, where `t <= 0`.
 * @param b The ending value, where `t >= 1`.
 */
public static function lerp(t:Float, a:Float, b:Float):Float;

/**
 * Computes the base-10 logarithm of a number.
 */
public static function log10(x:Float):Float;

/**
 * Computes the base-2 logarithm of a number.
 */
public static function log2(x:Float):Float;

/**
 * Returns the hyperbolic arcsine of a number.
 */
public static function asinh(x:Float):Float;

/**
 * Returns the hyperbolic arccosine of a number.
 */
public static function acosh(x:Float):Float;

/**
 * Returns the hyperbolic arctangent of a number.
 */
public static function atanh(x:Float):Float;

/**
 * Returns the hyperbolic sine of a number.
 */
public static function sinh(x:Float):Float;

/**
 * Returns the hyperbolic cosine of a number.
 */
public static function cosh(x:Float):Float;

/**
 * Returns the hyperbolic tangent of a number.
 */
public static function tanh(x:Float):Float;
```

The following refactors to existing mathematical functions (these have been changed to accept `haxe.Rest` as an argument, allowing them to accept three or more inputs at once).

```haxe
/**
 * Computes the minimum of all the listed floats.
 * NOTE: This would replace the existing `Math.min(a, b)` method to accept any number of arguments.
 * This would allow users to do `Math.min(a, b, c, d)` rather than `Math.min(Math.min(a, b), Math.min(c, d))`, which is error prone and hard to read.
 */
public static function min(...args:Float):Float;

/**
 * Computes the maximum of all the listed floats.
 * NOTE: This would replace the existing `Math.max(a, b)` method to accept any number of arguments.
 * This would allow users to do `Math.max(a, b, c, d)` rather than `Math.max(Math.max(a, b), Math.max(c, d))`, which is error prone and hard to read.
 */
public static function max(...args:Float):Float;
```

The following additional integer mathematical functions (these are suffixed with `i` to distinguish them from the original and return `Int` rather than `Float`, which conveniently prevents the user from having to call `Std.int()` on the result of every math function).

```haxe
/**
 * Computes the minimum of all the listed integers.
 * NOTE: Returning `int` prevents the need to cast the result.
 */
public static function mini(...args:Int):Int;

/**
 * Computes the maximum of all the listed integers.
 * NOTE: Returning `int` prevents the need to cast the result.
 */
public static function maxi(...args:Int):Int;

/**
 * Computes the value of the base raised to the specified exponent.
 * NOTE: Returning `int` prevents the need to cast the result.
 */
public static function powi(base:Int, exponent:Int):Int;

/**
 * Computes the absolute value of a number.
 * NOTE: Returning `int` prevents the need to cast the result.
 */
public static function absi(x:Int):Int;
```

## Impact on existing code

Haxe maintainers whould seek to implement the above changes in such a manner that they do not create compatibility issues, to maintain the functionality of existing code, as they are part of an important standard library.

## Drawbacks

Adding these new functions would requires development and maintenance of these functions for all of Haxe's supported platforms. Additionally, there may or may not be performance issues with the changes proposed to `min` and `max`. Additionally, the intent of the `Integer` function variations may not be clear without clearly written documentation.

## Alternatives

Users could write implementations for this functionality themselves, but as those implementations would not be part of the standard library, they would not be thoroughly tested across platforms.

## Unresolved questions

It is undetermined whether it is viable to replace `min` and `max` with ones utilizing `haxe.Rest`, or whether there are performance issues involved with doing so.
