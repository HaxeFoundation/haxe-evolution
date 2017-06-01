# Union types

* Proposal: [HXP-0000](0000-uniontypes.md)
* Author: [Mark Knol](https://github.com/markknol)

## Introduction

A union type describes a value that can be one of several types.

## Motivation

We have `haxe.ds.Either<L,R>` and `EitherType<T1,T2>` to allow some kind of dual type, but the syntax is a bit verbose and also limited to two types or externs only as far I'm aware of. I'd love to see it being changed to a nicer syntax and let it become part of the language. 

## Detailed design

#### Syntax

It can be uses with `|` to separate each type:

 * `Float|String` is the type of a value that can be a float or a string
 * `Float|String|Bool` is the type of a value that can be a float, string or a boolean.

It should be invalid to repeat a type multiple times: `Float|Float`

Union types should be usable as in various forms:

```haxe
// parameter
function foo(value:Float|String) { 
	
}

foo(3.14); // valid
foo("hey"); // valid
foo(true); // compile error: parameter 'value' should be Float or String

// return type
function qux(value):Float|String 
	return "great!"; // valid

function bar(value):Float|String  
	return 1.3; // valid

function baz(value):Float|String 
	return true; // compile error: function foo() should return Float or String

// typedef
typedef FloatOrString = Float|String;

// variables
var value:Float|String;
```

Let's say we have two types: `TypeA` and `TypeB`. If the both have a field with the same type, it should be possible to safely access the field without a specified cast. If a variable is present but the types are different, it will lead in a error.

```haxe
class TypeA {
	public var a:String = "any";
	public var b:Int = 5;
	
	public function new() {}
}
class TypeB {
	public var a:String = "a";
	public var b:String = "b";
	public var c:String = "c";
	
	public function new() {}
}


var value:MyType = TypeA|TypeB;
value.a; // valid 
// valid because TypeA.a and TypeB.a are string

value.b; // compile error: cannot unify field "b" of TypeA and TypeB
// invalid because 'TypeA.b' and 'TypeB.b' are of different types

value.c; // compile error: MyType has no field "c" (hint: cast to TypeB)
// invalid because only TypeB has a field 'c'
```

Union type should also be usable as type parameter constraint. 
```haxe
function foo<T:String|Float>(a:T) {
	
}
```
Again, here if valid fields are used, no cast is needed (uses TypeA and TypeB of previous example):
```haxe
function foo<T:TypeA|TypeB>(value:T):String {
	return value.a;
}
```

#### Union type switch

It would be great for to allow switching on these types. Not sure if this needs a different proposal. I'm also not sure if we need something in the pattern matcher, or need a new switch-like keyword, I leave that up to the team, since I cannot judge what is the best here. I know TypeScript uses `switch(value.kind)`.

Anyway, this is the _idea_:

```haxe
function foo(value:Float|String) {
	switch(typeof value) { // I'm open for any syntax here
		case Float: 
			trace("Float!" + value);
			
		case String:
			trace("String!" + value);
	}
}

```

It would translates to something like this. 

```haxe
function foo(value:Float|String) {
	if (Std.is(value, Float)) {
		 trace("Float!" + value);
	} else if (Std.is(value, String)) {
		 var value:String = cast value;
		 trace("String!" + value);
		 trace("String!" + value.toUpperCase());
	}
}
```

Since it does pattern matching, it can give errors like this:

```haxe
var value:Float|String
switch(typeof value) {
	case Float: 
	case String: 
	case Bool: // Compiler error: Invalid case Bool; is not Float or String
}
```

When the user didn't switch, then (s)he needs to do the cast her/himself:

```haxe
var value:Float|String;

if (Std.is(value, String)) {
	var a:String = cast value; // allowed, safe
}

var b:String = value; // Allowed, since it could unify to String
var c:Bool = value; // Error: cannot cast Float or String to Bool.
```

## Impact on existing code

* It shouldn't break existing code since this is an addition to the awesome type system we already have.
* It shouldn't break runtime too, since the type checking should be done compile time.
* I'm not sure if there can be issues with operator overloading "@:op(a | b) public function or(type1:Type, type2:Type)` with abstracts. In theory that could break existing code.

## Drawbacks

I can imaging using union types could cause performance hit on certain targets, since it is some kind of dynamic.
 
## Opening possibilities

This would make the type system richer. It should also make writing externs nicer.
I can imaging the union type switch can also very nice when working with `Any` / `Dynamic`.

## Unresolved questions

 * [ ] How should the syntax of the union type switch look like?
 * [ ] I'm not sure if one should be able to define something like `Class<String|Float>` or how to deal with that.
 * [ ] Is it possible to use such union types with abstracts: `abstract Bla(String|Float)` and if can one create functions like `@:to function foo(v:String|Float)`?
 
 
