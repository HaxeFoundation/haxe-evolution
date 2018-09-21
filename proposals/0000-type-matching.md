# Type matching

* Proposal: [HXP-0000](0000-type-matching.md)
* Author: [Alexander Kuzmenko](https://github.com/RealyUniqueName)

## Introduction

This proposal is an attempt to improve checking and casting a variable to another type.

## Motivation

It's a common case to check if a variable is of some type and cast it to that type.

<sub>This section of proposal showcases `if` examples only. See [Detailed design](#Detailed-design) section for other cases.</sub>

### Existing methods

There are two ways to do it in Haxe.

#### `Std.is()` and typed cast

```haxe
if(Std.is(developer, Indie) && cast(developer, Indie).hasMotivation()) {
	crowd.throwMoneyAt(cast(developer, Indie));
	cast(developer, Indie).createGameEngine();
}
```
Drawbacks:
* Performance penalty because both `Std.is()` and typed cast perform the same type checks every time.
* Verbose syntax.

#### `Std.is()` and untyped cast

```haxe
if(Std.is(developer, Indie) && (cast developer:Indie).hasMotivation()) {
	crowd.throwMoneyAt(cast indie);
	var indie:Indie = cast developer;
	indie.createGameEngine();
}
```
Drawbacks:
* Brings `Dynamic` because of untyped cast. Which can lead to all kinds of runtime errors, especially when it comes to refactoring.
* Verbose syntax if you need to use an API of checked type.

### Proposed method

```haxe
if(developer is indie:Indie && indie.hasMotivation()) {
	crowd.throwMoneyAt(indie);
	indie.createGameEngine();
}
```
Benefits:
* Better performance thanks to only one runtime type check.
* Completely type safe.
* Clean syntax.

## Detailed design

Type matching declares a new variable of checked type. This variable should only be available in branches of code which get executed if type matching evaluated to true.
```haxe
var isMotivatedIndie = developer is indie:Indie && indie.hasMotivation();
indie.createGameEngine(); //Error: unknown identifier "indie".

var isMotivatedIndie = !(developer is indie:Indie) && indie.hasMotivation(); // Error: unknown identifier "indie"
var isStudent = developer is student:Student || student.hasMotivation();     // Error: unknown identifier "student"
```

### Type matching as a condition for `if` and `while`

Declared variable is available inside of `if` condition and `if` block only if type check evaluated to `true`:
```haxe
if(developer is indie:Indie && indie.hasMotivation()) {
	indie.createGameEngine();
} else {
	indie.procrastinate(); //Error: unknown identifier "indie"
}
indie.createGame(); //Error: unknown identifier "indie"
```
Declared variable should not be available in branches of code which can be executed even if type matching evaluated to `false`
```haxe
if((developer is indie:Indie && indie.hasMotivation()) || anotherCondition()) {
	indie.createGameEngine(); //Error: unknown identifier "indie"
}

if(
	(developer is indie:Indie && indie.hasMotivation())
	|| indie.hasMotivation()  //Error: unknown identifier "indie"
) {
	indie.createGameEngine(); //Error: unknown identifier "indie"
}
```
The same rules apply to `while` syntax:
```haxe
while(developers[0] is indie:Indie && indie.hasMotivation()) {
	indie.createGameEngine();
	developers.shift();
}
indie.createGame(); //Error: unknown identifier "indie"

while(developers[0] is indie:Indie && indie.hasMotivation() || anotherCondition()) {
	indie.createGameEngine(); //Error: unknown identifier "indie"
	developers.shift();
}

do {
	indie.createGameEngine(); //Error: unknown identifier "indie"
	developers.shift();
} while(developers[0] is indie:Indie && indie.hasMotivation());
indie.createGame(); //Error: unknown identifier "indie"
```

### Type matching as a condition or a guard for `case` in `switch`

If type matching is used in `case` condition then left hand operand of `is` becomes a declared variable of checked type.
Variables introduced by type matching in `case` or in a guard are only available in a body or a guard of that `case`:

```haxe
switch(developer) {
	case newbie is Indie if(newbie.hasMotivation() && developer is student:Student):
		student.learnToCode();
		newbie.createGameEngine();
		//`indie` and `hired` are not available here
	case indie is Indie:
		indie.createGameEngine();
		//`student`, `newbie` and `hired` are not available here
	case hired is Hired:
		hired.resolveTask();
		//`student`, `newbie` and `indie` are not available here
}
//`student`, `newbie`, `indie` and `hired` are not available here
```

### Expression definition for `is` keyword.

```haxe
EIs(type:ComplexType, ?checked:Expr, ?declared:String);
```
Depending on a place where `is` keyword is used either `checked` or `declared` may be `null`.

## Impact on existing code

This proposal introduces new syntax without breaking old one. So it can only impact some rare macro code which tries to handle all kinds of expressions.

## Drawbacks

Can't imagine any drawbacks.

## Alternatives

The only alternatives that have meaning are described in [Motivation](#Motivation) section.
Trying to implement such behavior with macros looks like too complex task.

## Unresolved questions

* Is it allowed to mix type matching and structure matching in a single `switch`?
