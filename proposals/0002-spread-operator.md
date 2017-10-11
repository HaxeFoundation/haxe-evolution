# Feature name

* Proposal: [HXP-0002](0002-spread-operator.md)
* Author: [Thomas Fétiveau](https://github.com/zabojad)


## Introduction

Add ES6-like `...` spread operator to the Haxe syntax on both [arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator) and [objects](https://babeljs.io/docs/plugins/transform-object-rest-spread/).


## Motivation

Currently, there is a gap in Haxe for object duplication and overriding:

* A lot of very popular js libs like React or Redux require developers to implement [pure functions](https://en.wikipedia.org/wiki/Pure_function) massively and this operator makes it really easy to do and avoid over-verbosity of simple operations.
  
  This:
  ```
  function pure(data : MyStruct) {

  	return {...data, overwrittenFieldName: overwrittenFieldValue};
  }
  ```

  is clearer and more performant than this:
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

* The standard javascript externs do not expose the [Object.assign API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) that could be used in place of a spread operator on objects.

  While it's not complicated to write an extern class for this, it feels wrong to add it to the source code of a custom project while it is something that should be available by default.

* This `...` operator is part of the ECMAScript standard which Haxe syntax follows to some point. Getting closer to the ECMAScript syntax would ease the adoption of Haxe by javascript developers. 

* Also, if added to the Haxe syntax, we would also get several other benefits out of it:
```
var myArr = [ 1, 2, 3 ];
// array decomposition (non existent in Haxe and useful when calling externs)
myFunc(...myArr); // here, would do the same as: myFunc(1,2,3);
// shorter and more powerful array literals / comprehension
var myNewArr = [ 0, ...myArr, 4, 5 ];
trace(myNewArr); // [ 0, 1, 2, 3, 4, 5 ]

// object duplication
var myObj = { a: 3, b: 4 };
var myNewObj = { ...myObj }
// object duplication with field adding / overwriting
var myNewObj = { ...myObj, b: 7, c: 88 }
trace(myNewObj); // { a: 3, b: 7, c: 88 }
```
[More infos on spread operators with Objects](https://babeljs.io/docs/plugins/transform-object-rest-spread/).
[More infos on spread operators with Arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator).


## Detailed design

* We need to introduce a new unary operator e.g. OpSpread that's parsed from `...<expr>` (which is actually a breaking change for macros, since adding new enum constructor will affect exhaustiveness checks).

* `{ ...obj, field2: 42 }` could translate (depending on the targetted platform) to: 
By default:
```
{ field1: obj.field1, field2: 42 } // one thing I wonder here is how to avoid duplication of optional 
                                   // undefined fields. Do you see any way to avoid it ?
```

For the js target, we could do the following (it's what babel do):
```
// 1: define a global _extends function that has the same API as Object.assign
var _extends = Object.assign || function (target) { for (var i = 1; i < arguments.length; i++) { var source = arguments[i]; for (var key in source) { if (Object.prototype.hasOwnProperty.call(source, key)) { target[key] = source[key]; } } } return target; };

// 2: replace the `...` operator on objects with calls to this _extends function
_extends(obj, { field2: 42 });)
```

* `[ 1, 2, ...arr, 42, ...arr2, 1 ]` could translate to:
By default:
```
var tmp = [1,2];
for (v in arr) tmp.push(v);
tmp.push(42);
for (v in arr2) tmp.push(v);
tmp.push(1);
```

Or: (depending on what is more performant)
```
[ 1, 2 ].concat(arr).concat([ 42 ]).concat(arr2).concat(1);
```


For the js target, we could also probably output directly this:
```
[ 1, 2 ].concat(arr, [ 42 ], arr2, [ 1 ]) // concat method of js array allows 1+ arguments
```

* `aFunc(...myArr)` could translate to:
```
// for the js target, we could also probably output directly this:
aFunc.apply(thisRef,myArr);

// on other targets, I don't know...
```

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

It will also ease the addition of a es6+ target for Haxe in the future. [ES6 is almost 100% in all modern browsers (even Edge 14 is 93% ES6 compliant) and in Node.js](http://kangax.github.io/compat-table/es6/).


## Unresolved questions

Which parts of the design in question is still to be determined?

=> The request is based on an existing ES6 specification. What is left to be determined is more about how to implement it within Haxe itself.
