
# Integer data types

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [MJ](https://github.com/flashultra)

## Introduction

The current proposal have four goals:
- Replacing Int64 , UInt and UInt64 with native types (where is possible) .
- Allowing easy assignment for Int64 ,UInt and UInt64
- Introduce new type UInt64 for all Haxe targets and use native, where is possible.
- Allowing easy conversion between integer data types.


## Motivation

###  Replacing  Int64 , UInt and UInt64 with native types
Some of the Haxe targets support integer types natively.
  - C/C++ - -   int64_t,  uint32_t,  uint64_t;  
  - Java - long ; 
  - C# - long, uint, ulong
 
 Using native type for above targets which allowed could give better performance.
 
 For other targets:
 - Pyton - numbers  have unlimited size.
 - JS - could be use abstraction over BigInt ( for Int64 and UInt64) as it have better performance. 
 -  Php - by default integer is 64-bit ( for 64-bit systems) and 32-bits ( for 32-bit system). Haxe could emulate 64-bit (over 32-bit systems) and use default Int for others.
  - Lua (>=5.3) - support 64-bit integer types 
 
###  Allowing easy assignment for Int64 ,UInt and UInt64
Before Int64 literal suffix, it was more inconvenient to use Int64 (by calling Int64.make() ) and to create Int64 constant.
Still, sometimes is less readable when for example have this 
```haxe
    var x = 2147483i64;  // this is 2147483 ( Int64)
	var y = 2147483164;  // this is 2147483164 ( Int32)
```
Using a number separator makes things better, but still be better to use ```var y:Int64 = 4294967296;``` vs ```var y:Int64 = 4294967296_i64;``` as we know the type of the variable.
  At the current moment it's not possible to assign  value with decimal representation above 0x7FFFFFFF ( 2147483647) for UInt 
  For example,  the following assigment is invalid ``` var m:UInt = 3147483647;``` which is incovinient.
  For UInt64 should be allowed to assign above 9223372036854775807 ( 0x7FFFFFFFFFFFFFFF) in the code.

### Introduce new type UInt64 
At the moment  UInt64 is introduced in C#, C++ and Eval. Should be presented in all Haxe target as emulation or as native type ( if possible) 

### Allowing easy conversion between integer data types.
The current solution with the suffix give soltion for the problems like this 
```haxe
var mask:Int64 = 1i64 << 32;
```
but not for the problem like this:
```haxe
    var one:Int = 1;
    var mask:Int64 = one << 32;
```
The following code could be rewrite like this
```haxe
    var one:Int = 1;
    var mask:Int64 = Int64.make(0,one) << 32
```
or like 
```haxe
  var one:Int = 1;
  var mask = Int64.shl(one,32);
```
Still this won't give a solution for the problem of multiplying UInt 
```haxe
    var m:UInt = 0xBB9AC9FF;
    var n:UInt = 0xEC7A87C6;
    var mask:Int64 = m*n;
    //or
    var mask:Int64 = Int64.mul(m,n);
```
For different targets, give  dfferent result, but never correct one ```-5959250239385521094```
So if I want to multiply two UInt ( or Int) and want to receive as result Int64 is impossible in Haxe (or miss something?) . 
One solution will be to add prefix before operand for the expected type ( similiar as c/c++/java/c#) which will allow and easy implementation for those targets.
Example :
```haxe
    var m:UInt = 0xBB9AC9FF;
    var n:UInt = 0xEC7A87C6;
    var mask:Int64 = (i64)m*n;
    //or
    var mask:Int64 = (i64)m*(i64)n;
    var mask:Int64 = m*(i64)n;
```


## Detailed design

Many of the Haxe targets supports Int64 type ( long, long long int , Int64 ) , so where is possible the Int64 abstract class should be replaced with the native for the target, and this should give better performance too. 
This will match Haxe with many other languages where assignment on integer values is possible without calling function ( Int64.make)
 Also, it will be possible to use implicitly typed variables using this order against the value : ~~Int < UInt < Int64 < UInt64~~  ```Int < Int64 < BigInt```
For example: 
```haxe
  var x = 2147483647; // Int
  var y = 4294967295; // Int64  //~~UInt
  var v = 9223372036854775807; // Int64
  var w = 18446744073709551615; // BigInt  //~~UInt64
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

Easy conversion between different integer types could be implemented with adding preffix before the operand which will convert one int type to another. 
For example : i64, u32, ui64 ;  ```(i64)mask;```, ```(i32)mask;```
Using only numbers (at the end)  in the suffix and the prefix could lead to less readable code , so it could be added other alphabet such as: i64_t, u32_t, ui64_t ; ```(i64_t)mask;```, ```(u32_t)mask;```

Other question is how to convert UInt var to Int64 
For example if have:
```haxe
  var m:UInt = 0xBB9AC9FF;
  var mask:Int64 = Int64.make(0x0,m); //didn't work
```
So we want to keep Int64.high to 0x0 and assign the UInt var only the the low part.
It's not possible in Haxe at the moment. So something like this could be allow:
``` var mask:Int64 = (m & (i64_t)0xFFFFFFFF);```

## Impact on existing code

None

## Drawbacks

The problem will be with the implicitly vars if the user think assigment of the 4294967295 will lead to Int64, not UInt, so maybe the type should be mandatory for values above 2147483647 ?  It won't be a problem if usin only signed integer types ( Int, Int64, BigInt)


## Alternatives

Alternatives will be using Int64 literal suffix and number separator

## Opening possibilities


## Unresolved questions
