# Arrow functions

* Proposal: [HXP-0000](0000-arrow-functions.md)
* Author: [Alexander Kuzmenko](https://github.com/RealyUniqueName)

## Introduction

Provide better syntax for anonymous functions declaration.

## Motivation

* Even simple closures encourage splitting expressions into multiple lines, which makes code too bloated.
    It's hard to read and maintain code that heavily relies on anonymous functions.
    ```haxe
    //hard to read
    var names = users.map(function(u) return u.getProfile()).filter(function(p) return p.inCountry('USA')).map(function(p) return p.getName());

    //a little better
    var profiles = users.map(function(u) return u.getProfile());
    profiles = profiles.filter(function(p) return p.inCountry('USA'));
    var names = profiles.map(function(p) return p.getName());

    //best
    var names = users.map(u -> u.getProfile()).filter(p -> p.inCountry('USA')).map(p -> p.getName());
    ```

* In some cases total length of boilerplate code may be two times bigger than useful code size.
    ```haxe
    array.map(function(a) return a.toInt()).sort(function(a, b) return a - b);
    //vs
    array.map(a -> a.toInt()).sort((a, b) -> a - b);
    ```
    Required boilerplate reduced from 32 chars to 4.

* Nowadays arrow functions are largely adopted by programmers community.
    Lack of such functions in Haxe raises questions for newcomers that Haxe suffers stagnation in development.
    This factor has negative impact on growth of Haxe community.

* Current syntax of anonymous functions in Haxe is quite verbose compared to other languages.
    ```haxe
    //Haxe
    function() return expr
    function(arg) return expr

    //TypeScript
    () => expr
    arg => expr

    //JavaScript
    () => expr
    arg => expr

    //C#
    () => expr
    arg => expr

    //Scala
    () => expr
    (arg) => expr

    //Java
    () -> expr
    (arg) -> expr

    //Swift
    { expr }
    { arg1 in expr }

    //Ruby
    ->() {expr}
    ->(arg) {expr}

    //And so on...
    ```

## Detailed design

### Syntax

Following syntax is proposed for arrow functions in Haxe:

* No arguments:

    ```haxe
    () -> expr
    //equivalent for
    function() return expr;
    ```

* Single argument:

    ```haxe
    arg -> expr
    //equivalent for
    function(arg) return expr;
    ```

* Multiple arguments:

    ```haxe
    (arg1, arg2) -> expr
    //equivalent for
    function(arg1, arg2) return expr;
    ```

* Explicit typing:

    ```haxe
    (arg1:Int, arg2:String) -> expr
    //equivalent for
    function(arg1:Int, arg2:String) return expr;
    ```
    Explicit return type for arrow functions should not be allowed to not interfer with existing syntax of function type.

### AST

To distinct normal and arrow function syntaxes in AST new field `@:optional isLambda:Bool` is proposed to be added to [haxe.macro.Function](http://api.haxe.org/haxe/macro/Function.html) structure. Such change will have minimal impact on existing macros.

## Impact on existing code

Arrow functions will have no impact on existing code.

It's a compile-time feature so it will not affect runtime.

## Drawbacks

Since macros are being executed before typing, it can be hard to find out in macros whether arrow function should return something or not.  This can complicate transformations of arrow function by build macro if it needs to handle `return` expression or to insert something at the end of closure body.

## Alternatives

Since users got used to use arrow functions in other languages, various implementations were created with macros: at least two librarys on haxelib
and few more projects in outher sources.
However such implementations impact compilation time, introduce non-standart (and different) syntax and should be avoided inside of other macros (breaks compiler cache)
