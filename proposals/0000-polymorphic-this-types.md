# Polymorphic `this` types

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [mak](https://github.com/matulkum)

## Introduction

This feature is directly taking from Typescript. Therefore I am just quoting from the [Typescript docs](https://www.typescriptlang.org/docs/handbook/advanced-types.html):

"A polymorphic this type represents a type that is the subtype of the containing class or interface. This is called F-bounded polymorphism. This makes hierarchical fluent interfaces much easier to express (...)"

## Motivation

"Method chaining" is a very common programming model. Especially (but not only) in Javascript. When you extend a class which use this model you will encounter the problem, that calling a method from the subclass, which is implemented it its super class, will return the super classes type. This type does not have the methods which were defined in the sub class itself. This effectively "breaks the chain".

This is just one example where this feature would be useful. Surely there are others.

## Detailed design

Example:
```
class BasicCalculator {
    public var result = 0.0;

    public function new () {}

    public function add( a: Int): this {
        result += a;
        return this;
    }
}


class Calculator extends BasicCalculator {
    public function sin(): this {
        this.result = Math.sin(this.result);
        return this; 
           
    }
}

class Test {
    static function main() {
        var calculator = new Calculator();
        var result = calculator.add(2,3).sin().result; // <-- this would not be possible in current Haxe
    }
}
```

In current Haxe, if you would want to use "method chaining", you would have to cast, like so:
```
class BasicCalculator {
    public var result= 0.0;

    public function new () {}

    public function add( a: Int) {
        result += a;
        return this;
    }
}


class Calculator extends BasicCalculator {
    public function sin() {
        this.result = Math.sin(this.result);
        return this; 
           
    }
}

class Test {
    static function main() {
        var calculator = new Calculator();

        var result = cast(calculator.add(2), Calculator).sin().result; <-- cast, because calculator.add() returns BasicCalculator"
        trace(result);
    }
}
``` 

## Impact on existing code

None.

## Drawbacks

None.

## Alternatives

None.

## Unresolved questions

None?
