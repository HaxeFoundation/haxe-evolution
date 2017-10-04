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

## Motivation

Anonymous structures and their `typedef`d variants are the most straightforward way to interoperate with native objects on most dynamic targets, like JavaScript, Lua, Python, etc.

However it has one serious flaw: if the object contains a field name that is a reserved Haxe keyword, it cannot be described and accessed without use of reflection. This makes code less descriptive, unnecessarily ugly and potentionally less type-safe.

## Detailed design

From the user's perspective it's simply about adding `@:native("fieldName")` metadata to the anonymous structure definition, as shown in the example above.

However care should be taken not to lose `@:native` information between structure unifications, so some design is required in that part.

When unifying anonymous structures, we add additional check for `@:native` metadata on fields with the following rules:

 * let `f1` and `f2` be fields of two structures being unified with the same name,
 * do the usual kind and visibility checks
 * let `n1` and `n2` be the optional string values of `@:native` metadata of these fields
 * if both `n1` and `n2` are absent, fields unify
 * if both `n1` and `n2` present and have the same value, fields unify
 * if both `n1` and `n2` present, but have different value, fields don't unify
 * if either of `n1`, `n2` is present while another one is absent, fields don't unify

This will ensure that it's safe to pass values of different anonymous structure types with `@:native` is involved.

Another place where additional handling is required is object declarations. We want to be able to create objects with `@:native` fields, so, when the expected type of an object declaration expression is a known structure, we relax the `@:native` unification rule above and allow creating the object using specified field names:

```haxe
// TODO: have to go right now
```

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
