# `@:native` for structure fields

* Proposal: [HXP-NNNN](NNNN-anon-native.md)
* Author: [Dan Korostelev](https://github.com/nadako)

## Introduction

Support `@:native` metadata on anonymous structure fields, so it's possible to use structure field names that are reserved keywords in Haxe.

```haxe
typedef JsonSchema = {
  @:native("enum") var enum_:Array<JsonValue>;
}

schema.enum_ = [1,2,3]; // generates schema.enum = [1,2,3]
print(schema.enum_); // generates print(schema.enum)
```

Related issues and discussions:
 * https://github.com/HaxeFoundation/haxe/issues/5105
 * https://github.com/HaxeFoundation/haxe/pull/2185

## Motivation

Anonymous structures and their `typedef`d variants are the most straightforward way to interoperate with native objects on most dynamic targets, like JavaScript, Lua, Python, etc.

However it has one serious flaw: if an object contains a field name that is a reserved Haxe keyword (e.g. `enum`) or an invalid Haxe identifier (e.g. `min-width`), it cannot be described and accessed without use of reflection. This makes code less descriptive, unnecessarily ugly and potentionally less type-safe.

## Detailed design

From the user's perspective it's simply about adding `@:native("fieldName")` metadata to the anonymous structure definition, as shown in the example above. So `@:native` has similar/same semantics like it does in extern classes.

However care should be taken not to lose `@:native` information between structure unifications, so some constraints are required in that part.

When unifying anonymous structures, we add additional check for `@:native` metadata on fields with the following rules:

 * let `f1` and `f2` be fields of two structures being unified with the same name
 * do the usual kind and visibility checks
 * let `n1` and `n2` be the string values of optional `@:native` metadata of these fields
 * if both `n1` and `n2` are absent, fields unify, proceed to type checks
 * if both `n1` and `n2` present and have the same value, fields unify, proceed to type checks
 * if both `n1` and `n2` present, but have different values, fields don't unify, emit an error
 * if either of `n1`, `n2` is present while another one is absent, fields don't unify, emit an error

This will ensure that it's safe to pass values of different anonymous structure types when `@:native` is involved.

Another place where additional handling is required is object declarations. We want to be able to create objects with `@:native` fields, so, when the expected type of an object declaration expression is a known structure, we relax the `@:native` unification rule above and allow creating the object using specified field names. In the generated code the field in the object declaration should be renamed to `@:native` version:

```haxe
var f:{ @:native("final") var isFinal:Bool; } = { isFinal: true };
f.isFinal = false;
trace(f.isFinal);
```

Should generate:

```haxe
var f = { final: true };
f.final = false;
trace(f.final);
```

One might be surprised that this is allowed, while the following code will produce a "@:native mismatch" error (as per rules described above):

```haxe
var o = { isFinal: true };
var f:{ @:native("final") var isFinal:Bool; } = o;
```

But this is similar to how `@:structInit` and `@:from macro` is handled and should be easy to communicate. Also, for the most of actual use-case scenarios (passing objects to extern functions), this will be minor to non-issue.

This should provide a straightforward and safe way of representing these kind of fields without introducing hidden costs.

## Impact on existing code

At the moment, `@:native` on structure fields is not handled in any way, so this should not break any existing behaviour unless there's some particular macro processing, but given the semantics `@:native` has on classes, I don't think this meta is used for anything other than the proposed feature.

I know just one [macro implementation](https://github.com/clemos/haxe-js-kit/blob/develop/util/NativeMap.hx) that uses `@:native` on structures to provide the field renaming feature by @clemos that is used like [this](https://github.com/clemos/haxe-js-kit/blob/6447e9f07a430836a7f4f5eafd2f8f71895639ea/js/atomshell/browser/BrowserWindow.hx#L15-L35). With the addition of the proposed feature, the macro shouldn't break, but will become obsolete and could be removed, I think.

## Drawbacks

I don't see any immediate drawbacks to this approach. It doesn't forfeit safety while providing the desired feature in the most obvious way that is requested by users every now and then.

## Alternatives

An alternative would be to have a macro that builds an abstract that forwards field access by different names through generated get/set properties. But I think it's insane to have that much engineering for the desired feature, and it's nothing a normal user would like or know how to do in the first place.
