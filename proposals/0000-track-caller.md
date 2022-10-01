# Caller tracking.

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Zeta](https://github.com/Apprentice-Alchemist)

## Introduction

`@:trackCaller` metadata + `haxe.Magic.callLocation()` as a replacement for `?pos:haxe.PosInfos`.

## Motivation

The magic `?pos:haxe.PosInfos` argument does not work with rest arguments.

## Detailed design

```hx
@:trackCaller
function foo() {
	trace(haxe.Magic.callLocation());
}

foo();

// turns into

function foo(caller:haxe.PosInfos) {
	trace(caller);
}

foo(haxe.Magic.currentLocation());
```
When rest arguments are involved, the exact implementation depends on how rest arguments are implemented.

When `@:trackCaller` is applied to interface methods, abstract methods or methods that will be overriden,
the implementors/overriders inherit the attribute.

`var closureOfTrackCallerFun = foo` turns into `var closureOfTrackCallerFun = foo.bind(haxe.Magic.currentLocation())`.

`haxe.Magic.callLocation()` returns the position of the last call of a `@:trackCaller` function in a non-trackcaller functions.

For example:
```hx
@:trackCaller
function bar() {
	foo();
}

@:trackCaller
function foo() {
	trace(haxe.Magic.callLocation());
}

// turns into

function bar(caller: haxe.PosInfos) {
	foo(caller);
}

function foo(caller:haxe.PosInfos) {
	trace(caller);
}
```

## Impact on existing code

None.

## Drawbacks

None.

## Alternatives

The magic `?pos:haxe.PosInfos` argument could be made even more magic by being allowed after Rest arguments.

## Opening possibilities

## Unresolved questions

Where to put the `callLocation` function.
