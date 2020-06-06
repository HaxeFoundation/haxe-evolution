# Void as unit type

* Proposal: [HXP-NNNN](NNNN-unit-void.md)
* Author: [Dan Korostelev](https://github.com/nadako)

## Introduction

Allow using `Void` in value places, similarly to unit types in other languages (e.g. `unit` in OCaml or `Unit` in Kotlin, `Void` in Swift, `NoneType` in Python, `()` in Rust and Haskell, `monostate` in C++):

```haxe
class Signal<T> {
	function trigger(payload:T);
}

var signal = new Signal<Void>();
signal.trigger(Void);
```

## Motivation

By supporting type parameters, Haxe allows for creating generic reusable "building block" types, such as `Either<L,R>`, `Signal<T>`, `Future<T>`, `Callback<T>` etc. However due to the lack of the [unit type](https://en.wikipedia.org/wiki/Unit_type) in standard library, there is no standard way to use such parametrized types when no actual value is expected (e.g. a `Signal` that carry no payload).

What library and codebase authors usually do to circumvent this is introducing their own non-standard unit types (e.g. [`tink.core.Noise`](https://github.com/haxetink/tink_core/blob/master/src/tink/core/Noise.hx), [`thx.core.Nil`](https://github.com/fponticelli/thx.core/blob/master/src/thx/Nil.hx), [vshaxe's `NoData`](https://github.com/vshaxe/vscode-json-rpc/blob/2ab29247ed8848a6f505095f6cf1a27f7389d27a/src/jsonrpc/Types.hx#L111), [`haxe.NoData` from #9111](https://github.com/HaxeFoundation/haxe/pull/9111/files#diff-0d9c638c36b82754b8f819195267cabb)), which is unfortunate because it hurts interoperability and unit type is generally a very basic thing that should be provided by the language itself.

## Detailed design

In this proposal I suggest extending the existing `Void` type with the functionality of unit type because it already has the closest meaning to the unit type.

This means that the `Void` identifier in expression place can be used as a value for the `Void` type, like in the introductionary example:

```haxe
class Signal<T> {
	function trigger(payload:T);
}

var signal = new Signal<Void>();
signal.trigger(Void);
```

At run-time, the `Void` value can be platform-specific, and the type is completely opaque. On most targets it can be a native `null`/`nil`/`undefined` value.

### Void arguments

Haxe already allows `Void` arguments in function signatures, however trying to call such functions will emit the `Cannot use Void as value` error. This restriction should be lifted to allow passing the singleton `Void` value.

### Void return

We do NOT change the behaviour of `Void` return types: functions still return no value and can be generated as native void-functions. This should work because Haxe already handles the case where `Void`-returning function is passed as a callback (e.g. JVM provides an overload that returns `null`).

Explicit `return Void` can be transformed to no-value `return` by the compiler.

When a known `Void`-returning call is used in a value place, the compiler can generate `{voidReturn(); Void;}`, however whether that is required depends on what we decide to do with `Void` in value places in general.

### Void in known value places

Currently there are some compiler errors specific to `Void`, such as:
 - `Variables of type Void are not allowed`
 - `Cannot use Void as value`
 - `Void should be Dynamic`

These restrictions make sense, so we could keep them, while only allowing to pass `Void` as an argument. Alternatively we could lift these restrictions (except maybe the `Dynamic` unification) and treat `Void` as a normal unit type.

### Void unification

Nothing changes here, `Void` can only be unified with `Void`.

## Impact on existing code

There should not be any of impact on the existing code, since we're allowing what was previously forbidden.

One tricky subject is what to do with the old function type notation `Void->T`. Currently that is inconsistent with the all the other types and will resolve into zero-arity function type (in modern notation: `()->T`). Now if we allow Void as unit type, the `Void->T` starts to make some sense in new function notation as an _unary_ function that takes `Void` unit type, however it should be a very rare case, because functions that take `Void` arguments don't make much sense outside of type parameter cases.

I would suggest leaving this as-is for now and start actively discourating usage of the old function types so we can eventually remove it and make `Void->T` an unary function.

## Drawbacks

I'm not aware of any drawbacks here so far. One thing I can think of is that the name `Void` can be a bit confusing, but that's just the state of unit type in [current programming languages](https://en.wikipedia.org/wiki/Unit_type#In_programming_languages) in general.

## Alternatives

An alternative that is worth considering is adding a separate unit type to the standard library, like it is done in [#9540](https://github.com/HaxeFoundation/haxe/pull/9540), however I feel like it's better to reuse the existing `Void` type for this. I think adding a separate type for this brings more confusion and has worse ergonomics (especially if one has to import it).

## Opening possibilities

This change seems pretty self-contained.

## Unresolved questions

As said, I'm not sure if we should lift all Void in value places restrictions for the sake of consistency or leave them because they seem to make sense for sane code.
