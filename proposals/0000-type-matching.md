# Type matching

* Proposal: [HXP-0000](0000-type-matching.md)
* Author: [Alexander Kuzmenko](https://github.com/RealyUniqueName)

## Introduction

This proposal is an attempt to improve checking and casting a variable to another type.

## Motivation

It's a common case to check if a variable is of some type and cast it to that type.

<sub>This section of proposal showcases `if` examples only. See [Detailed design](#Detailed-design) section for `switch` examples.</sub>

### Existing methods

There are already two ways to do it in Haxe.

#### `Std.is()` and typed cast

```haxe
if(Std.is(developer, Indie) && (cast developer:Indie).hasMotivation()) {
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
	var indie:Indie = cast developer;
	crowd.throwMoneyAt(indie);
	indie.createGameEngine();
}
```
Drawbacks: 
* Brings `Dynamic` because of untyped cast. Which can lead to all kinds of runtime errors, especially when it comes to refactoring.
* Verbose syntax if you need to use an API of checked type.

### Proposed method

```haxe
if((developer is indie:Indie) && indie.hasMotivation()) {
	crowd.throwMoneyAt(indie);
	indie.createGameEngine();
}
```
Benefits: 
* Better performance thanks to only one runtime type check.
* Completely type safe.
* Clean syntax.

## Detailed design

### Type matching as a condition for `if` and `while`



Describe the proposed design in details the way language user can understand
and compiler developer can implement. Show corner cases, provide usage examples,
describe how this solution is better than current workarounds.

## Impact on existing code

What impact this change will have on existing code? Will it break compilation?
Will it compile, but break in run-time? How easy it is to migrate existing Haxe code?

## Drawbacks

Describe the drawbacks of the proposed design worth consideration. This doesn't include
breaking changes, since that's described in the previous section.

## Alternatives

What alternatives have you considered to address the same problem, why the proposed solution is better?

## Opening possibilities

Does this change make other future changes possible or easier? Leave this section out if the proposed change
is completely self-contained.

## Unresolved questions

Which parts of the design in question is still to be determined?
