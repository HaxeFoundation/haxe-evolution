# Integer data types

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [MJ](https://github.com/flashultra)

## Introduction

Allowing easy assignment of Int64 type for all Haxe targets. Replacing with native Int64  , where is it possible and use abstract for others.

## Motivation

Before Int64 literal suffix, it was more inconvenient to use Int64 (by calling Int64.make() ) and to create Int64 constant.
Still, sometimes is less readable when for example have this 
```haxe
    var x = 2147483i64;  // this is 2147483 ( Int64)
	var y = 2147483164;  // this is 2147483164 ( Int32)
```
Using a number separator makes things better, but still be better to use ```var y:Int64 = 4294967296;``` vs ```var y:Int64 = 4294967296_i64;``` as we know the type of the variable.

## Detailed design

Many of the Haxe targets supports Int64 type ( long, long long int , Int64 ) , so where is possible the Int64 abstract class should be replaced with the native for the target, and this should give better performance too. 
This will match Haxe with many other languages where assignment on integer values is possible without calling function ( Int64.make)
 Also, it will be possible to use implicitly typed variables using this order against the value : ```Int < UInt < Int64 < UInt64```.
For example: 
```haxe
  var x = 2147483647; // Int
  var y = 4294967295; // UInt
  var v = 9223372036854775807; // Int64
  var w = 18446744073709551615; // UInt64
```
The new UInt64 type should be available for all targets ( at the moment, it is introduced in cs,cpp,eval).
Of course, the following assignment should be possible too:
```haxe
  var x:Int = 2147483647;
  var y:UInt = 4294967295;
  var v:Int64 = 9223372036854775807;
  var w:UInt64 = 18446744073709551615;
```
The other thing is Int and Int32. For some target Int could be a float ( javascript, for example), so not having a consistent overflow could lead to strange results and expectations. Not sure how many use Int and unexpected recieve Int64 value after that. Maybe it will be good to unify those two classes, but not sure how this could affect performance ( for javascript , php , python, Lua) 
For example ``` var b:Int  = 2147483647 * 349342;``` will give 750206232210274 in JS ( 0x2AA4EFFFAAB62)

## Impact on existing code

None

## Drawbacks

The problem will be with the implicitly vars if the user think assigment of the 4294967295 will lead to Int64, not UInt, so maybe the type should be mandatory for values above 2147483647 ?

## Alternatives

Alternatives will be using Int64 literal suffix and number separator

## Opening possibilities


## Unresolved questions


