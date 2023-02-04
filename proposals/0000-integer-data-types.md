
  
# Integer data types

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [MJ](https://github.com/flashultra)

## Introduction

The current proposal have four goals:
- Replace Int64, UInt and UInt64 with native types (where possible) .
- Allows easy assignment for Int64, UInt and UInt64.
- Introduce new UInt64 type for all Haxe targets and use native where possible..
- Allows easy conversion between integer data types.


## Motivation

###  Replacing  Int64 , UInt and UInt64 with native types
Some of Haxe's targets support integers natively.
  - C/C++ - -   int64_t,  uint32_t,  uint64_t;  
  - Java - long ; 
  - C# - long, uint, ulong
 
 Using native types, for above targets, could give better performance.
 
 For other targets:
 - Pyton - numbers  have unlimited size.
 - JS - could be use abstraction over BigInt ( for Int64 and UInt64) as it have better performance. 
 -  Php - by default integer is 64-bit ( for 64-bit systems) and 32-bits ( for 32-bit system). Haxe could emulate 64-bit (over 32-bit systems) and use default Int for others.
  - Lua (>=5.3) - support 64-bit integer types 
 
###  Allowing easy assignment for Int64 ,UInt and UInt64

**Int64**
Before Int64 literal suffix, it was more inconvenient to use Int64 (by calling Int64.make() ) and to create Int64 constant.
Still, sometimes is less readable when, for example, have this 
```
	var x = 2147483i64;  // this is 2147483 ( Int64)
	var y = 2147483164;  // this is 2147483164 ( Int32)
```
Using a number separator makes things better, but still be better to use ```var y:Int64 = 4294967296;``` vs ```var y:Int64 = 4294967296_i64;``` as we know the type of the variable.

**UInt**
  It is currently not possible to assign a decimal value greater than 0x7FFFFFFF (2147483647) to a UInt.
  For example, the following assignment is invalid ``` var m:UInt = 3147483647;``` and requires the use of a hexadecimal value (0xBB9AC9FF), which is inconvenient.
  
  **UInt64** 
 For UInt64, assignment above 9223372036854775807 ( 0x7FFFFFFFFFFFFFFF) should be allowed  directly in source code.

### Introduce new type UInt64 
At the moment  UInt64 is introduced in C#, C++ and Eval. Should be introduced in all Haxe target as emulation or  native type ( if possible) 

### Allowing easy conversion between integer data types.
The current implementation of the suffix provides a solution to problems like this:
```haxe
// Created mask 0000 0001 0000 0000
var mask:Int64 = 1i64 << 32; // 0x100000000 = 4294967296
```
but not for the problem like this:
```haxe
    var one:Int = 1;
    var mask:Int64 = one << 32; // will return 1
```
The above code can be rewritten like this:
```haxe
    var one:Int = 1;
    var mask:Int64 = Int64.make(0,one) << 32; // = 0x100000000 
```
or so
```haxe
  var one:Int = 1;
  var mask = Int64.shl(one,32);
```
Both ways require using Int64's internal methods. 
A cleaner and easier solution would be to use suffix conversion:
```haxe
    var one:Int = 1;
    var mask:Int64 = (i64_t)one << 32; //convert the one variable to Int64 and do a left shift
```
However, using Int64 internal methods, won't provide a solution  for   UInt multiplication 
```haxe
    var m:UInt = 0xBB9AC9FF;
    var n:UInt = 0xEC7A87C6;
    var mask:Int64 = m*n; // return 298038336 , which is Int type
    //or
    // For js, eval give 375817154890806330
    // Crash for Hashlink last development release
    var mask:Int64 = Int64.mul(m,n);  //correct value should be -5959250239385521094
    
```
It returns the wrong result in Haxe, not the correct one:
```-5959250239385521094```
So if I want to multiply two UInts and want to get an Int64 as a result, it's impossible in Haxe.
One solution would be to add a prefix before the operand for the expected type (similar to c/c++/java/c#), which would also allow easier implementation for those targets.
Example :
```haxe
    var m:UInt = 0xBB9AC9FF;
    var n:UInt = 0xEC7A87C6;
    var mask:Int64 = (i64)m*n; //convert m to Int64 and return Int64 after multiplication
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

Easy conversion between different integers can be implemented by adding a prefix before the operand that will convert one int type to another.
For example : i64, u32, ui64 ;  ```(i64)mask;```, ```(i32)mask;```

Using numbers (at the end) of the suffix and prefix can result in less readable code, so another letter can be added like:  i64_t, u32_t, ui64_t ; ```(i64_t)mask;```, ```(u32_t)mask;```

Another question is how to convert a UInt variable to Int64 without to accept the first bit as a negative number to Int64
For example, if I have:
```haxe
  var m:UInt = 0xBB9AC9FF;
  var mask:Int64 = Int64.make(0x0,m); //didn't work
  // I want to have 
  // mask = (0x00000000 , 0xBB9AC9FF )
```
So we want to keep Int64.high at 0x0 and assign UInt var only the low part.
This is not currently possible in Haxe. So something like this might be allowed in the future:
``` var mask:Int64 = (m & (i64_t)0xFFFFFFFF);```

## Impact on existing code

None

## Drawbacks

The problem will be with the implicitly vars if the user think assigment of the 4294967295 will lead to Int64, not UInt, so maybe the type should be mandatory for values above 2147483647 ?  It won't be a problem if using only signed integer types ( Int, Int64, BigInt)


## Alternatives

Alternatives will be using Int64 literal suffix and number separator

## Opening possibilities


## Unresolved questions

### **1) How to convert from one integer type to another?**

There are at least two possible options:
- **Using а prefix** - Some of the lanuages use this as a solution. For example , C/C++ use  long long,  Java/C# uses  long . Sot it is possible to introduce new prefix in Haxe, like  (i64) , (i64_t) , (long) 
Example:
```
var one:Int = 1;
var  mask:Int64 = (i64)one<<32; //using prefix (i64)
```
- **Implicit conversion based on the declared type**
For example:
```
var one:Int = 1;
var  mask:Int64 = one<<32; // will return 0x100000000
```
As the ```mask``` expects Int64, the above code will convert ```one``` to Int64 and the result will be  4294967296 (0x100000000) .
But if we have:
```
var one:Int = 1;
var mask = one<<32; //result is 1, as mask is Int
```
The var ```mask``` will be converted to Int ( since we didn't specify the type and the other two operands are Int) and the result will be 1.
This solution is much clearer and replaces the need for a prefix with a declaration of the type of the result variable.
### **2) Conversion from Float to Int**
The division operation of two integers would return a Float .
In Haxe that is true for the Int type, but division of Int64 will return Int64 .
To convert to int the result of two int operands we should use ``Math.floor()`` as we not have a prefix ```(int)```,```(long)``` or implicit conversion between types.
 One solution could be to use type promotion similiar to C , where if operand on the right hand side is of lower rank type then it becomes the type of left hand side operand  as it have  higher rank.
 
A hierarchy of this type can be: ```Int < UInt < Int64 < UInt64 < Float```

For example , if we have:
```haxe
    var operand1 = 57;
    var operand2 = 6;
    var result:Int = operand1/operand2; // it will return 9
    
    // Same for Int64 
    var operand1:Int64 = 57_i64;
    var operand2:Int64 = 6_i64;
    var result:Int64 = operand1/operand2; // 9 
```
### **3) How to convert from signed to unsigned type and vice versa?**
At the current moment conversion from Int to Int64 ( using ```Int64.ofInt()```) is sign-extended , so if you want to keep Int64 without sign we should use ```Int64.make(0x0, <my_int_number>) ``` or a mask ```Int64.and(<my_int_number>,Int64.make(0x0,0xFFFFFFFF))``` . The other way will be to convert Int to UInt and after that to Int64 . 

Maybe a new prefix could be introduce for converting Int to Int64 ( no sign-extended), which will be similiar to UInt to Int64 conversion.

Example , if have variable n = -6, we should have :
| Let n=-6 | Int | UInt      | Int64 <br> (sign-extended) | Int64 <br> (no sign-extended) | UInt64 | Float |
|----------|-----|-----------|---------------------|------------------------|--------|-------|
|Int       | -6  | 4294967290|   -6                  |     4294967290                   |   4294967290     |   -6.0    |

### **4) Int vs Int32**
For C,C++,Java,C# the Int type hас  expected overflow behavior. For others, like  Javascript, Php, Python  , the numeric type defaults to 64-bit (or no limit), which leads to wrong results when the user expects an overflow.
So this will for example return 1 (or 0 if the static analyzer is on) for C/C++,Java,C#,Javascript, but 4294967296 in php (for 64bit system),Python.
```haxe
 var mask:Int = 1 <<32;
```
Note: javascript returns a correct value for bitwise operations

Another example will return a different value for a javascript :
```haxe
var m:Int = 2147483647*2; 
```
It will return 4294967294 for Javascript , Python, Php  and  -2 for C,C++,Java and C#
This inconsistency between targets could lead to unexpected results. Possible solutions could be:
- Keeping things as they are now - separate Int and Int32 classes
- Set Int to have the same behavior as Int32.
This can lead to a performance loss (in javascript and possibly in python and php) for heavy arithmetic operations, but such operations will probably expect proper overflow behavior and use Int32 anyway.
