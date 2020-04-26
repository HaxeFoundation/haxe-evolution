# Multiple arguments for array access

* Proposal: HXP-NNNN
* Author: [rikoo](https://github.com/ErikRikoo)

## Introduction

Introduce an extension to the array access operator in order to allow multiple parameters between []: `array["foo", 0]`.
This proposal also concern operator overloading.
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
- Using a function call with multiple arguments. Drawbacks:
  - Semantically () has not the same meaning that [].
  - Currently there is no way to overload the = operator so we can't do `a(0, 1) = "foo"`.
- We can use a chained array access. Drawback:
  - At runtime we need to add a lot more logic or return proxy objects to be able to use this syntax. It adds complexity and overhead to the code. 

## Detailed design 
**Grammar change:**
As stated above, it needs a change in the grammar.
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

**Syntax for operator overload**
I propose a new way to provide the @:op() arguments for array access.
We keep the same syntax:
```
@:op([]) 
public function arrayRead(n:Int) {}

@:op([]) public function arrayWrite(n:Int, value:Int) {}
```
But we extend it be able to use multiple arguments:
```
@:op([_, _])
public function arrayRead(n1:Int, n2:Int) {}

@:op([_, _])
public function arrayWrite(n1:Int, n2:Int, value:Int)
```


## Impact on existing code
**Grammar change**
- Option 1:
Because it is a new Enum value added, previous macro code will still be able to parse simple array access. It may break if they are using the default case in a switch on the expression definition.
- Option 2:
Because we add an optionnal argument to the definition the previous code will still match even if there is multiple arguments. Moreover, they will not be detected in the default case.

**Syntax of the overload operator**
The new syntax should not conflict with the old one.

## Drawbacks


## Alternatives

- Multiple array access that add more complexity or overhead.
- Function call that does not allow some basic operation such as using = operator, or it needs macros.

## Opening possibilities

## Unresolved questions

