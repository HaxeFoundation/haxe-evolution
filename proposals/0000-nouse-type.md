# A type for meaningless values

* Proposal: [HXP-0000](0000-nouse-type.md)
* Author: [Aleksandr Kuzmenko](https://github.com/RealyUniqueName)

## Introduction

Unlike `Void`, which implies no value at all, `NoUse` type would allow to express a value with no meaning in places where a value has to exist.
```haxe
class Signal<T> {
	function trigger(payload:T);
}

var signal = new Signal<NoUse>();
signal.trigger(null);
```
Typical solution to such needs in type theory is a [unit type](https://en.wikipedia.org/wiki/Unit_type) (a type with a single possible value).
But for convenience we want that type to behave as similar to `Void` as possible. And `Void` in haxe unifies with any type if used in function return type position:
```haxe
var fn:()->Void = function():Int return 0; //this is allowed
```
So the proposed `NoUse` type cannot be a type of a single value because at runtime it could be any value, but at compile time in Haxe's type system it would denote a meaningless or useless value. That is, the only usage of a value of type `NoUse` is to be passed around.

## Motivation

By supporting type parameters, Haxe allows for creating generic reusable "building block" types, such as `Either<L,R>`, `Signal<T>`, `Future<T>`, `Callback<T>` etc. However due to the lack of the unit type in standard library, there is no standard way to use such parametrized types when no actual value is expected (e.g. a `Signal` that carry no payload).

What library and codebase authors usually do to circumvent this is introducing their own non-standard unit types (e.g. [`tink.core.Noise`](https://github.com/haxetink/tink_core/blob/master/src/tink/core/Noise.hx), [`thx.core.Nil`](https://github.com/fponticelli/thx.core/blob/master/src/thx/Nil.hx), [vshaxe's `NoData`](https://github.com/vshaxe/vscode-json-rpc/blob/2ab29247ed8848a6f505095f6cf1a27f7389d27a/src/jsonrpc/Types.hx#L111), [`haxe.NoData` from #9111](https://github.com/HaxeFoundation/haxe/pull/9111/files#diff-0d9c638c36b82754b8f819195267cabb)), which is unfortunate because it hurts interoperability and unit type is generally a very basic thing that should be provided by the language itself.

## Detailed design

```haxe
enum abstract NoUse(Null<Dynamic>) from Dynamic {
	var NoUse = null;
}
```
We want any type in function return type position to unify with `NoUse` hence `from Dynamic` part.

*For simplicity all further examples will imply usage of this API:*
```haxe
typedef Action<T> = ()->T;
typedef Callback<T> = (outcome:T)->Void;

/** Do something and pass the result to the callback */
function produce(action:Action<T>, callback:Callback<T>):Void;
/** Perform an action which does not produce a useful outcome */
function perform(action:Action<NoUse>, ?callback:Callback<NoUse>):Void;
```

### Binding a monomorph to Void

Binding a monomorph to `Void` should produce a compilation error.
```haxe
produce(function():Void {}, _ -> {}); //Error: cannot use Void as a value
```

### Automatic generation of `return` expressions

Compiler could automatically add `return null` expression to function with `NoUse` return type:

```haxe
//no explicit `return null` expression is required
perform(() -> trace('Hello'));
```

However that does not mean using functions with `Void` return type is allowed in such cases because `Void` imply no value unlike `NoUse`
```haxe
function voidFn():Void;

perform(voidFn); // Error: ()->Void should be ()->NoUse
```
In such cases users should explicitly wrap `voidFn` in another function:
```haxe
perform(() -> voidFn()); //Ok if auto-generation of `return` expressions is implemented.
```

## Impact on existing code

Currently `Void` is allowed as type parameter. [Binding a monomorph to Void](#Binding-a-monomorph-to-Void) section suggests to disallow that which would be a breaking change. Though that's not mandatory and just aims to make the type system a bit saner.

## Drawbacks

Proposed solution is not a well-known unit type, but a slightly different concept with a different name. Which reduces discoverability of the feature by new users.

## Alternatives

An alternative would be to use a "classic" unit type with a single value. But that means `performUnit(returnsInt)` won't be allowed while `performVoid(returnsInt)` is allowed while both expressions have the same meaning:
```haxe
function performVoid(fn:()->Void);
function performUnit(fn:()->Unit);
function returnsInt():Int;

performVoid(returnsInt); //ok
performUnit(returnsInt); //Error: `Int should be Unit`
```
Also classic unit type would allow to detect if a dynamic value is of unit type because such type could exist at runtime if implemented via normal `enum` or `class`:
```haxe
Std.isOfType(dynamicValue, Unit); // true
```

## Unresolved questions

1. `from Dynamic` allows to pass any value in any place, not only unification of return types. Though it fits the description of `NoUse` type. So maybe it's ok.
2. Automatic generation of `return` expressions is controversial. It may add confusion to code reading making some callbacks to look like `(smthng)->Void` while they are actually `(smthng)->NoUse`. On the other hand it doesn't make much difference on semantics and it's just annoying to have to put `return null` expressions just to satisfy the compiler with no practical meaning. Especially if you have to do it in an expression with a lot of branches (`switch` or `if...else if` trees).