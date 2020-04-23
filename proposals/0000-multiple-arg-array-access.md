# Multiple arguments for array access

* Proposal: HXP-NNNN
* Author: [rikoo](https://github.com/ErikRikoo)

## Introduction

Introduce an extension to the array access operator in order to allow multiple parameters between []: `array["foo", 0]`.
This only proposes a grammar and AST modification to parse it in a macro. But it doesn't concern operator overloading (for now...).
This proposal follows my [issue](https://github.com/HaxeFoundation/haxe/issues/9339).

## Motivation

This feature proposes, as stated above, is to provide multi arguments array access. 
For now, we can only have one argument and if we provide mulitple we get an error.
Exemple:
```
array1[0] // allowed
array2[0, "foo"] // Get an "Expected ]" error
```

**Workarounds:**
- We could use a function call with multiple arguments but the purpose would be less clear than the array access. 
  If there is an assignement operator (=, +=, ...) after, it would be even less cleat at the first sight.
- We could also use chained array access but it would be less direct to use it in macro time.
  Moreover, it doesn't provide the same meaning:
  Multiple Access with one argument means "one access provide some value used to get another one".


## Detailed design
As stated above it is just a change in the grammar to allow multiple arguments in an array access like in a function.
I propose to add a new enum value in Expr.ExprDef:
```
enum ExprDef {
   ...
   EArray(e1:Expr, e2:Expr); // We keep this one for retro compatibility

   EArrayMultiple(e1:Expr, params:Array<Expr>);
   ...
}
```
Then when a `array[a, b...]` is in the code it will be put in the AST and it can be matched during compile time.
The different targets can then throw an exception if it is still there when they get the AST.
Or they could also generate code that use multiple argument array access when it is allowed like in C# or python.

## Impact on existing code

Because it is a new Enum value added, previous macro code will still be able to parse simple array access.
It may break if they are using the default case in a switch on the expression definition.

## Drawbacks


## Alternatives

I thought about other syntax that can be parsed like multiple array access or function call that can be parsed.
But it is less clear or more tricky to parse in a macro.

## Opening possibilities

With this change to grammar it opens to overload of array access with multiple argument array access.
I think it could be a first step to that.

## Unresolved questions

