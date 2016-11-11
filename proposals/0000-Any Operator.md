# Any Operator Overloading, by use of couching ``

* Proposal: [Any Operatator ( Any String For Abstract Operators ) ](Any Operators.md)
* Author: [Justinfront](https://github.com/justinfront)

## Introduction

Allow abstracts to use any custom string as an operator by couching the strings with ``

## Motivation

Abstract allow custom operators but the symbols that can be used for these operators are extremely limited. 
Abstracts are ideal for creating Mathematical libraries to help encaptulate complex mathematics and physics, 
both in comercial games and education and scientific research, think Haxe nice replacement for fortran ;).
But with only a few symbol Operators it is hard to create expressive abstracts that line up with current science notation.
For instance you may want a library that provides a cross product operator "X".
My suggestion is to couch any operator string that a user wants within ``

To allow abstracts operators to be more expressive, and thus attract more University type interest.

## Detailed design

I propose the use of ` to couch operators so that any string could be used, so for instance users could create an `add` function.

Usage:
```haxe
abstract MyAbstract(String) from String to String {
  public inline function new(s:String) {
    this = s;
  }

  @:op(A `add` B)
  public function add(rhs:Int):MyAbstract {
    return new MyAbstract(this + rhs);
  }
}

class Main {
  static public function main() {
    var a = new MyAbstract("operator");
    trace( operator `add` ' flexibility' );
  }
}
```

This should be implemented to work with binary, unary, and comparisom opperators.
So users could use suitable abstracts with unary and comparisom to create more natural code.
``` haxe
var next = `inc` stepper;
var value = ( fred `faster than` tom );
```

## Impact on existing code

Not sure there would be any?

## Drawbacks

It's a bit ugly and verbose, but seems like a feasible work round.

## Alternatives

I think ` seems the best wrapper since I am not aware it's really used elsewhere apart from within strings.

## Opening possibilities

It would allow more expressive Maths libraries that would not have additional runtime overhead, 
and would provide more interest to Science and Maths community and perhaps get Haxe used by more students, 
and thus more future generations of Haxers.
