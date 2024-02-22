# Caller tracking.

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Zeta](https://github.com/Apprentice-Alchemist)

## Introduction

`@:trackCaller` metadata + `haxe.Magic.callerLocation()` as a replacement for `?pos:haxe.PosInfos`.

## Motivation

The magic `?pos:haxe.PosInfos` argument does not work with rest arguments.

## Detailed design

```hx
@:trackCaller
function foo() {
	trace(haxe.Magic.callerLocation());
}

foo();

// turns into

function foo(caller:haxe.PosInfos) {
	trace(caller);
}

foo($currentLocation);
```
When rest arguments are involved, the exact implementation may be different (possibly depending on how they are implemented on the target).

`var closureOfTrackCallerFun = foo` turns into `var closureOfTrackCallerFun = foo.bind($currentLocation)`.

`haxe.Magic.callerLocation()` returns the position of the last call of a `@:trackCaller` function in a non-trackcaller functions.
This behaviour could be then be turned off by using `@:trackCaller(noPropagate)`.

For example:
```hx
@:trackCaller
function bar() {
	foo();
}

@:trackCaller
function foo() {
	trace(haxe.Magic.callerLocation());
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

Walking up the stack in `callerLocation` is also a possibility, but would break when inlining is involved.
It also requires generation of some kind of debug info.

Make the compiler allow `callerLocation` in default arguments eg `pos:haxe.PosInfos = callerLocation()`.
Would break in the case `(pos:haxe.PosInfos = callerLocation(), ...rest:haxe.PosInfos)`.

Make `@:callerLocation` meta on default argument eg `@:callerLocation pos:haxe.PosInfos = cast null`.
Would break in the case `(@:callerLocation pos:haxe.PosInfos = cast null, ...rest:haxe.PosInfos)` unless magic semantics are given to the default argument when annotated with `@:callerLocation`.

## Opening possibilities

Turn `trace` into `@:trackCaller(noPropagate) function trace(...args:Dynamic)`.
Then the only magic left would be auto-importing it everywhere.

## Unresolved questions

`haxe.Magic` is a bit too magic. Where should the `callerLocation` function live? A module-level function in `haxe.PosInfos`?

How this would work with interfaces and abstract/overriden methods.

Is `noPropagate` the best name to disable propagation? Maybe `@:trackCaller(ignorePropagation)` is better? 