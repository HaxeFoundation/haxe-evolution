# ::Method Overloading::

* Proposal: [HXP-NNNN](0000-Method-Overloading.md)
* Author: [David Mouton](https://github.com/damoebius)

## Introduction

Method Overloading is a feature that allows a class to have two or more methods having same name, if their argument lists are different.

Argument lists could differ in –

1. Number of parameters.

2. Data type of parameters.

3. Sequence of Data type of parameters.

## Motivation

It's a usefull feature in Java and C#, and overloading with externs is realy comfortable.

## Detailed design

In simple terms we can say that a class can have more than one methods with same name but with different number of arguments or different types of arguments or both.

```javascript
public function start():Void {
        trace('0');
}

public function start(mode:String):Void {
        trace('1');
}

public function start(end:Bool):Void {
        trace('2');
}
```

## Impact on existing code

None because it's a new feature.

## Drawbacks

I don't know any drawbacks

## Alternatives

Maybe using Union type and optionnal parameters. But method overloading is more readable and we doesn' need to test which paramaters are used.
Using Abstract is a current alternative : see http://www.kevinresol.com/2016-11-23/haxe-method-overloading/

But i'm talking about readability and maintnability and using abstract like this is uncomfortable.

## Opening possibilities

## Unresolved questions
We already talked about it in 2012 : https://groups.google.com/forum/#!topic/haxelang/feujdbvrrrQ
I hope things have changed
