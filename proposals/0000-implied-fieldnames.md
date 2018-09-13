# Implied Field Names on Object Literals

* Proposal: [HXP-NNNN](0000-implied-fieldnames.md)
* Author: [Neil Graham](https://github.com/Lerc)

## Introduction

Add implied field names to object literals. 

If a field name and value is represented by the same text allow it to be written just once.

Essentially allow `{fish,cheese}` instead of requiring `{fish:fish,cheese:cheese}`

## Motivation

This allows simplification of code, making it easier to read and
reducing the possibility of errors.

It is common practice to have variables that will be placed into a
field of an object that shares the name of the variable.  
Allowing the simplified form is not only more concise and easier to read, but also
highlights instances of the expanded form as being notably different.

```
var x=1.5;
var y=3;
var pt2 = {x, y}
var pt3 = {x, y, z:someOtherValue}     // z stands out as a special case
```

In addition to the reasons here, It is worth noting that this feature has been 
added to Javascript and most of the justifications for doing so also apply here
as well as the additional aspect of easing transition between JavaScript 
and Haxe programming.



## Detailed design

For any object literal field definition the value may be omitted if the identifier
specifying the field name is also a valid identifier for an in-scope value;

This could also be thought of the other way around,  If an object literal value is represented by a single identifier, the field name need not be specified and will take the name of the identifier.

Thus for any object that may be specified as
```
var example = {a:a,  b:b, c:d, e:a+b};
```
could also be specified as 
```
var example = {a,b, c:d, e:a+b};
```

This would only occur in instances where the value can be specified by a single identifier.

## Impact on existing code

This should keep compatibility with existing code.   No changes would be required, but a
great deal of existing code could be rendered smaller and more readable by utilizing this
feature.  A parser could easily identify opportunities and recommend this change 

## Drawbacks

undiscovered.

## Alternatives

The alternative is just to live without it, but that does require typing identifier:identifier a lot which always has the possibility of introducing errors by typo.

## Opening possibilities

Recent versions of ECMAScript have introduced a full suite of enhancements that bear looking at,  Not simply to mimic JavaScript, but because the reasoning behind those enhancements is quite sound.   

This proposal is the simplest of those enhancements, I intent to propose more of these enhancements for discussion in future.  Should this proposal succeed I would propose looking at the destructuring assignment enhancements next;
