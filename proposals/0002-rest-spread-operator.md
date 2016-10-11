# Feature name

* Proposal: [HXP-0002](0002-rest-spread-operator.md)
* Author: [Thomas Fétiveau](https://github.com/zabojad)


## Introduction

Add ES6-like `...` spread and rest operator to the Haxe syntax on both [arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator) and [objects](https://babeljs.io/docs/plugins/transform-object-rest-spread/).


## Motivation

Currently, there is a gap in Haxe for object duplication and overriding:

* A lot of very popular js libs like React or Redux require developers to implement [pure functions](https://en.wikipedia.org/wiki/Pure_function) massively and this operator makes it really easy to do and avoid over-verbosity of simple operations.
  
  This:
  ```
  function pure(data : MyStruct) {

  	return {...data, overwrittenFieldName: overwrittenFieldValue};
  }
  ```

  is much less verbose than this:
  ```
  function pure(data : MyStruct) {

  	var ret : MyStruct = {};

  	for (f in Reflect.fields(data)) {

  		Reflect.setField(ret,f,Reflect.field(data,f));
  	}

  	ret.overwrittenFieldName = overwrittenFieldValue;

  	return ret;
  }
  ```

* The standard javascript externs do not expose the [Object.assign API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign).

  While it's not complicated to write an extern class for this, it feels wrong to add it to the source code of a custom project while it is something that should be available by default.

* This `...` operator is part of the ECMAScript standard which Haxe syntax follows to some point. Getting closer to the ECMAScript syntax would ease the adoption of Haxe by javascript developers. 

* Also, if added to the Haxe syntax, we would also get several other benefits out of it:
```
// array decomposition (non existent in Haxe)
myFunction(...myArray);
// shorter and more powerful array literals / comprehension
var myNewArr = [ 1, 4, ...myOldArr, 56 ];
// even augmented push
myOldArr.push(...myNewArr);
// object duplication
var myNewObj = { ...myOldObj }
// object duplication with field adding / overwriting
var myNewObj = { ...myOldObj, overwrittenField: v, newField: y }
```


## Detailed design

There are two cases to differentiate when using the `...` operator:

* [On Objects](https://babeljs.io/docs/plugins/transform-object-rest-spread/):
```
// Spread properties
var n = { x, y, ...z };
trace(n); // { x: 1, y: 2, a: 3, b: 4 }

// Rest properties
var { x, y, ...z } = { x: 1, y: 2, a: 3, b: 4 };
trace(x); // 1
trace(y); // 2
trace(z); // { a: 3, b: 4 }
```

We gain a powerful object duplication operator that is missing in Haxe right now.


* On Arrays:

```
// Spread elements
var myArr = [ 1, 2, 3 ];
myFunc(...myArr);  // here, would do the same as: myFunc(1,2,3);
var anotherArr = [ 0, ...myArr, 4, 5 ]; // clear syntax for array literals
```
[More documentation on spread operator with Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator).

We gain an array decomposition feature that is missing in Haxe right now.

Plus, we also open new ways of manipulating arrays that helps in getting clearer and less verbose code.

```
// Rest elements
var [x, ...y] = ['a', 'b', 'c']; // x='a'; y=['b', 'c']
```
[More documentation on rest operator with Arrays](http://exploringjs.com/es6/ch_destructuring.html#sec_rest-operator).


## Impact on existing code

There should not be any impact on existing Haxe code as this would be an additional Haxe syntax feature. Nothing would be changed to other Haxe syntax details.


## Drawbacks

I'm not sure about this one.

Drawbacks may appear with type checking. Spreading the elements of an array should not prevent type checking but is it possible to type check the spreading of objects properties?


## Alternatives

This operator addresses several things like object duplication and code verbosity.

There isn't really a workaround for this request that adresses all this at once.


## Opening possibilities

This kind of changes opens more possibilities in the "marketing" of the language within communities like the javascript community.


## Unresolved questions

Which parts of the design in question is still to be determined?

=> The request is based on an existing ES6 specification. What is left to be determined is more about how to implement it within Haxe itself.
