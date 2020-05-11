# Typed Metadata

* Proposal: [HXP-NNNN](NNNN-typed-metadata.md)
* Author: [Mario Carbajal](https://github.com/basro)

## Introduction

Add new metadata syntax `@>path.to.Type()` that allows using imported types as metadata.

```haxe
import my.lib.MyMeta;
import haxe.meta.Keep;
import my.lib.Keep as MyKeep;

@>MyMeta
class Foo {
	@>Keep function foo() {}
	@>MyKeep function boop() {} // Name collisions are not a problem
	@>my.lib.MyKeep function mop() {} // Full paths also work
	@>RandomMetaName function bar() {} // Error RandomMetaName is unknown
}
```

## Motivation

Metadata in haxe is currently just a string. Most of the time this is needlessly dynamic.

This has the following drawbacks:
* Users are not notified if they make a typo or if the metadata they are using is deprecated.
* When making a macro library one must take care to use metadata names that will not collide with other libraries.
* Haxe can break existing code by adding new compiler metadata.
* If a name collision does happen the user is unable to resolve this.

Supporting typed metadata would fix all of these.

It would also provide a nice pathway to better completion support of library metadata.


## Detailed design

### Syntax

The proposed syntax is the same as current compile-time metadata except `>` is now allowed after `@`.

Same as with compile-time metadata the parser would generate MetadataEntry with it's name field including the `>` character.

For example `@>MyMeta(1)` would have MetadataEntry name `">MyMeta"`

### Type checking

During type checking the compiler will resolve the metadata entry names that start with `'>'` as if they were a type path.

Like so:
```haxe
static function getMetadataType(metadata: MetadataEntry) : Null<Type> {
	if ( metadata.name.charAt(0) == ">" ) {
		return Context.getType(metadata.name.substring(1)).follow();
	}
	return null;
}
```

Additionally at this stage the compiler could check for metadata in the resolved type. For example:
* Check that the resolved type has `@>haxe.meta.MetadataType`. This would make metadata types an opt in feature for extra type safety.
* Issue a warning if the type is marked as deprecated.

A `type: Null<Type>` field could be added to `MetadataEntry` that the compiler will fill with the resolved type.

Alternatively a new type can be introduced `TypedMetadataEntry` that extends `MetadataEntry` by adding the `type: Null<Type>` field. Then `EMeta` can have `MetadataEntry` while `TMeta` would have the typed one.

### Usage

You can define a metadata type by annotating another type with `@>haxe.meta.MetadataType`

```haxe
package my.lib;

@>haxe.meta.MetadataType
abstract MyMeta(Any) {}
```

Macros wishing to use typed metadata would then need to first type the metadata entry and then match the type.

For typing the metadata entry a `typeMetadata(meta):TypedMetadataEntry` method could be added to `Context` analog to `typeExpr`

Example macro:
```haxe
static function getMetaTypeId(meta:MetadataEntry) {
	var type Context.typeMetadata(meta).type;
	return
		if ( type == null ) null
		else type.getID(); // using tink macro getID()
}

static macro function build() : Array<Field> {
	var fields = Context.getBuildFields();

	for ( field in fields ) {
		var meta = Lambda.find(
			field.meta,
			meta -> getMetaTypeId(meta) == "my.lib.MyMeta"
		);
		if ( meta != null ) {
			// ... do something to the field ...
		}
	}
	return fields;
}
```

Macros that run in later stages of compilation would have access to the `TypedMetadataEntry` so they wouldn't need to call `Context.typeMetadata`

## Impact on existing code

No existing code is affected.

## Drawbacks

Macros will need to run type resolution in order to match the metadata, this is bound to be slower than the current string matching.

## Alternatives

Can't think of any.

## Opening possibilities

Metadata types could themselves have metadata that enable compiler features.

For instance make `@>haxe.meta.BuildField` metadata that makes the augmented metadata type run a build macro on a single class field. The metadata type itself could have a `static macro function buildField(field:Field):Field` that gets called by the compiler.

## Unresolved questions

* Mechanisms for non compile time typed metadata. I haven't used RTTI metadata so I didn't dare to figure that part out. I think typed metadata can start of as compile time only and add the RTTI parts later on.

* Perhaps instead of supporting any type as typed metadata, a new type specific for metadata could be introduced. Since I can't really think of a reason to ever use anything other than an abstract class as typed metadata.
