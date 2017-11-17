# Stacktypes : Stack-based Tuples

* Proposal: [HXP-0004](0004-stack-based-tuples.md)
* Author: [Justin Donaldson](https://github.com/jdonaldson)

## Introduction

Stack types include new syntax and new expressions for dealing with
references on stacks.

## Motivation

Stacks are used in most programming languages in order to pass arguments into
functions.  Haxe supports this usage natively, with additional extern support
for features such as Rest, Either, etc.

However, some languages use stacks as return values, and in iteration contexts.

e.g.
```lua
 for k,v in tbl : print(k,v) end
```

If we reference stack-based tuples with a conventional type in Haxe it causes
all kinds of problems.  @nadako and I worked through some of these for lua
multireturns, but it's not a first class solution.  Multireturn types cannot
function as type parameters, and there's currently no Haxe syntax capable of
generating them (extern only feature).

This problem also crops up with iteration, since many languages support
iteration over multiple values (python, etc.)


## Detailed design

The proposal is to introduce a new expression type representing a stack.  This
would commonly be used in certain iteration contexts, such as key/value:

```haxe
for ((k,v) in map){
  trace('$k: $v');
}
```

It would also be possible to generate these stack tuples as return types.
Languages like Lua could support these natively.  Other languages could support
these with the help of dummy container values (that are reusable within
threads).

```haxe
 (var x,var y,var z) = tupleGenerator();
// or maybe
 var (x,y,z) = tupleGenerator();
```

Stack-based tuples are not just for key/value iteration on certain targets.
Function argument expressions are also effectively the same thing.  This tuple
pattern would work well with coroutines that have more than one argument for
their callbacks:

```haxe
var some_async_function(cb : Bool->Int->Void){

};
var (success : Bool ,value : Int) = some_async_function().async;
```

The paren-based wrapper syntax clearly differentiates stack based datastructures
from other common Haxe expressions.  It also mirrors the existing syntax for
argument expressions These stack datastructures can *not* be used interchangeably with
normal Haxe types, since Haxe types all pertain to a single value/reference.
We'd need to make a few changes to the AST to accomodate this.


It should be possible to define a new type of typedef for stack types:
```haxe
stack StackType = (foo : Int, bar : Float, baz : String);
```

Stacktypes can be used as return types for functions, and you can use them to
extract a single value from the return.

```haxe
var functionReturningStackType() {return (foo: 1, bar : 1.1, baz : 'hi')} : StackType;
var foo = functionReturningStackType().foo;
```

Languages that support this natively could just use their existing mechanism
(e.g. Lua multireturn).  Languages that do not support it would provide an
array/anon wrapper for the values.


## Impact on existing code

This change will reuse old syntax  for new purposes.  The existing function
argument syntax becomes the shared basis for stack types.

Anonomous object expressions will need their field expressions to have a
consistent order.  This would involve a lot of refactoring in the compiler, but
I don't believe it would break anything.

This new change could be a breaking change depending on whether or not it is
implemented in the standard library.  Map iterators could be optimized with
stack types.  Compatibilty could be achieved by moving the currently affected
tyupes to hx3compat.

Migration to the new feature is straightforward.  It would involve adapting to
the new iteration style, and implementing stack types for both async/coroutine
operations and for multireturn scenarios.



## Drawbacks

This approach doesn't *really* introduce completely new syntax, but it is
reusing syntax in a new way, which means changes to the lexer and the AST.

## Alternatives

This approach is related to a large number of ongoing proposals/issues. Here's a
list:

Stacktypes could be used to assign into existing variable definitions,
supporting destructured assignments:
https://github.com/HaxeFoundation/haxe/issues/4300

Stacktypes could be used for maximally efficient key/value iteration:
https://github.com/HaxeFoundation/haxe-evolution/pull/37

Stacktypes could be used to generate a series of asynchronous values defined
through a continuation mechanism:
https://github.com/nadako/haxe-coroutines

Stacktypes could replace the awkward multireturn syntax used by Lua:
https://haxe.org/manual/target-lua-multireturns.html




## Opening possibilities

???

## Unresolved questions

Typedef syntax, integration with coroutines, integration with default key/value
iteration

