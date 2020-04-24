# Multiple arguments for array access

* Proposal: HXP-NNNN
* Author: [rikoo](https://github.com/ErikRikoo)

## Introduction

Introduce an extension to the array access operator in order to allow multiple parameters between []: `array["foo", 0]`.
This only proposes a grammar and AST modification to parse it in a macro. But it doesn't concern operator overloading (for now...).
This proposal follows my [issue](https://github.com/HaxeFoundation/haxe/issues/9339).

## Motivation

**Current state** 

This feature proposes, as stated above, to provide multi arguments array access. 
For now, we can only have one argument and if we provide mulitple we get an error.

Exemple:
```
array1[0] // allowed
array2[0, "foo"] // Get an "Expected ]" error
```

**Goals**

- I stepped into that issue when I tried to implement a multi dimensionnal array lib that would use a syntax close to numpy element indexing.
But we can imagine the same problem for any object that have a meaning for indexing with multiple access, for exemple a map with a pair of keys.

- With the proposed syntax we now have a more clear semantic on what we are accessing at the first sight and it improves code readability.
When using multiple array access, there could be conflicts between getting one element from the variable and then getting something from it and an access with multiple arguments.


**Workarounds:**

- We could use a function call with multiple arguments but the purpose would be less clear than the array access. 
  If there is an assignement operator (=, +=, ...) after, it would be even less cleat at the first sight.
- We could also use chained array access but it would be less direct to use it in macro time.
  Moreover, it doesn't provide the same meaning:
  Multiple Access with one argument means "one access provide some value used to get another one".


## Detailed design 
As stated above it is just a change in the grammar to allow multiple arguments in an array access like in a function.
I propose two possible changes:
- Option 1: **add a new ExprDef**
```
enum ExprDef {
   ...
   EArray(e1:Expr, e2:Expr); // We keep this one for retro compatibility

   EArrayMultiple(e1:Expr, params:Array<Expr>);
   ...
}
```

- Option 2: **changing EArray definition**
```
enum ExprDef {
   ...
   EArray(e1:Expr, e2:Expr, ?others:Array<Expr>);
   ...
}
```

Then when a `array[a, b...]` is in the code it will be put in the AST and it can be matched during compile time, like this:
```
  myMacro(array[0, "foo"]);
```

And the macro code:
- For Option 1:
```
  public static macro function myMacro(e:Expr):Expr {
    switch(e.expr) {
      case EArrayMultiple(e1, args):
        return generateExpr(e1, args);
      default:
        return null;
    }
  }
```
- For Option 2:
```
  public static macro function myMacro(e:Expr):Expr {
    switch(e.expr) {
      case EArray(e1, e2, args):
        return generateExpr(e1, e2, args);
      default:
        return null;
    }
  }
```

The different targets can then throw an exception if it is still there when they get the AST.
Or they could also generate code that use multiple argument array access when it is allowed like in C# or python.

## Impact on existing code
- Option 1:
Because it is a new Enum value added, previous macro code will still be able to parse simple array access. It may break if they are using the default case in a switch on the expression definition.
- Option 2:
Because we add an optionnal argument to the definition the previous code will still match even if there is multiple arguments. Moreover, they will not be detected in the default case.

## Drawbacks


## Alternatives

I thought about other syntax that can be parsed like multiple array access or function call that can be parsed.
But it is less clear or more tricky to parse in a macro.

## Opening possibilities

With this change to grammar it opens to overload of array access with multiple argument array access.
I think it could be a first step to that.

## Unresolved questions

