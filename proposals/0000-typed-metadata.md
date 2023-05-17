# Typed Metadata

* Proposal: [HXP-0000](0000-typed-metadata.md)
* Author: [Robert Borghese](https://github.com/RobertBorghese)

&nbsp;
&nbsp;

# Introduction

A typing system for Haxe metadata to validate a its arguments
and optionally provide compile-time transformations.

```haxe
// -----------------------
// mypack/Meta.hx

/**
	Define an author for a type definition.
**/
@:metadata function author(name: String): haxe.macro.Expr.TypeDefinition;

/**
	Transform an `Expr` to execute only if the `input`
	expression is not `null`.
**/
@:metadata function ifSome(input: haxe.macro.Expr): haxe.macro.Expr {
	final content = switch(Context.getDecoratorSubject().type) {
		case DExpression(e): e;
		case _: throw "Impossible";
	}
	return macro {
		final _t = $input;
		if(_t != null) {
			$content;
		}
	}
}

// -----------------------
// MyClass.hx

import mypack.Meta;

@author("Anonymous")
class MyClass {
	public function printPlus3(a: Null<Int>): Int {
		@ifSome(a) {
			trace(a + 3);
		}
	}
}
```

&nbsp;
&nbsp;

# Motivation

Metadata can be a little tedious to work with sometimes.

When making a program involving Haxe metadata, it's simple to check for
a specific meta's existance; however, things become way more complicated
when working with its arguments. A good amount of boilerplate needs
to be written to check if an argument exists, if it's the desired type,
and then to convert the `Expr` object into the desired format.

On the other hand using a Haxe library or framework that processes
metadata can become troublesome, as there is no guarentee the metadata
is documented properly. Not to mention there is no scoping control with
metadata, so there may be naming conflicts.

Typing metadata using function declarations provides a better format
for finding, documenting, and error checking Haxe metadata.

&nbsp;
&nbsp;

# Detailed design

There's a lot to cover here. A table has been provided for your convenience: 

| Topic | Description |
| --- | --- |
| [Basic Rules](0000-typed-metadata.md#basic-rules)                       | The basic syntax and rules for a metadata "function" declaration. |
| [Haxe API Changes](0000-typed-metadata.md#haxe-api-changes)             | The changes to the Haxe API required. |
| [New Metadata ](0000-typed-metadata.md#new-metadata)                    | List of new metadata used to configure metadata functions. |
| [Allowed Argument Types](0000-typed-metadata.md#allowed-argument-types) | List of argument types allowed for a typed metadata. |
| [Allowed Return Types](0000-typed-metadata.m#allowed-return-types)      | List of return types allowed for a typed metadata. |
| [Decorators](0000-typed-metadata.md#decorators)                         | The design of metadata that runs code from its function to modify its subject. |

## Basic Rules

A metadata can be declared using a function declaration with the `@:metadata` meta.

Metadata functions are allowed to have no function body, though they can use one
if desired. This will be covered later (see [Decorators](0000-typed-metadata.md#decorators)).
```haxe
@:metadata function myMeta(): Any;
```

### Metadata Function Restrictions

The `@:metadata` meta may only be used on module-level or static functions.
Furthermore, the `macro`, `dynamic`, `extern`, and `inline` flags cannot be used with a metadata function.
```haxe
@:metadata var myVar: Int; // error: @:metadata can only be used on static functions.
@:metadata macro function myMeta(): Any; // error: Invalid access on metadata function.
```

Metadata functions cannot be called normally. Any attempt to call them should
result in an error:
```haxe
function main() {
	myMeta(); // error: Function marked with @:metadata can only be used as metadata.
}
```

Type parameters are not allowed on metadata functions.
```haxe
@:metadata function myMeta<T>(); // error: Type parameters disalloweed on metadata functions.
```

### Metadata Scoping/Importing

This metadata can be used on anything that allows metadata on it currently.
However, it follows the same scoping rules as Haxe functions. Meaning it must
either be used with its full path (packages.Module.funcName) or be imported:
```haxe
@mypack.MyModule.myMeta function doThing() { ... }
```
```haxe
import mypack.MyModule;
@myMeta function doThing() { ... }
```

### Untyped Metadata

If the Haxe compiler encounters a metadata entry it cannot type, how it is
handled is an "Unresolved Question". 

For the time being, this proposal suggests throwing an error unless the
define `-D allow-untyped-meta` is defined.

### Metadata Target

The return type of the metadata function declaration dictates where this metadata
can be used. The `Any` type denotes a metadata can be used anywhere. Another example
is `haxe.macro.Expr` restricts a metadata's usage to expressions.

A full list of allowed return types can be found at [Allowed Return Types](0000-typed-metadata.md#allowed-return-types).
Any return type besides the ones listed there are not allowed and should result in an error.

```haxe
// use this anywhere
@:metadata function anyMeta(): Any;

// only allowed on expressions
@:metadata function exprMeta(): haxe.macro.Expr;

// error: Type `Int` is not valid type for metadata function.
@:metadata function intMeta(): Int;
``` 
### Basic Meta Arguments

Arguments can be added to the metadata functions. Like with return types, there are only
certain types allowed. A full list can be found at [Allowed Argument Types](0000-typed-metadata.md#allowed-argument-types).

Besides the argument type restriction, there are no other restrictions for arguments.
Optional arguments, default arguments, and rest arguments should work as normal.
```haxe
@:metadata function oneNum(num: Int): Any;
@oneNum(123) function doThing() { ... }

// ---

@:metadata function maybeNum(num: Int = 0): Any;
@maybeNum function doThing() { ... }
@maybeNum(123) function doThing2() { ... }

// ---

@:metadata function numAndStr(?num: Int, str: String): Any;
@numAndStr(123, "test") function doThing() { ... }
@numAndStr("test") function doThing2() { ... }

// ---

@:metadata function numRest(...num: Int): Any;
@numRest function doThing() { ... }
@numRest(1) function doThing2() { ... }
@numRest(1, 2, 3) function doThing3() { ... }

// ---

// error: Type `haxe.Exception` is not valid argument type for metadata function.
@:metadata function invalidType(o: haxe.Exception): Any;
```

&nbsp;
&nbsp;

## Haxe API Changes

A new optional field should be added to `haxe.macro.Expr.MetadataEntry`.

If this metadata entry is typed, then this field `field` will be filled with a
reference to the `ClassField` of the metadata function.
```haxe
// Unresolved question: would it be possible to have this typed?
// Maybe this should be haxe.macro.Expr.Field instead?
var ?field:Ref<haxe.macro.Type.ClassField>;
```

### Reading Arguments

There needs to be a mechanism for reading the arguments of a metadata.
To provide this, new class should be added: `haxe.macro.MetadataEntryTools`.

This class provides static extension functions for `MetadataEntry` that read
a specific argument type. For example, for the `Int` argument type, there should
be a `getInt(index: Int)` function that looks like this:

```haxe
static function getInt(entry: MetadataEntry, index: Int): Null<Int> {
	return if(entry.params != null && index < entry.params.length) {
		switch(entry.params[index].expr) {
			case EConst(CInt(v)): Std.parseInt(v);
			case _: null;
		}
	} else {
		null;
	}
}
```

There should be one function for every possible argument type.

These functions should not throw any errors, as that is the job of the Haxe compiler
on typed metadata. Instead `null` is returned if the argument doesn't exist or match
the desired type. Technically, these could also be used on untyped metadata.

### Function Type Struct

The following anonymous structure should be added to the `haxe/macro/Expr.hx` module:
```haxe
typedef FunctionPath = {
	> TypePath,
	functionName: String;
};
```

This is an untyped structure for storing type paths to functions. It is used as an
argument type for metadata. Long story short, it allows for type paths that end with
a lowercase identifier (`myFunc`, `Module.Sub.myFunc`).

Technically, function paths could be stored in `TypePath`, but they shouldn't. 

&nbsp;
&nbsp;

## New Metadata

There needs to be a way for metadata functions to configure a couple options:
 * Whether the metadata can be used multiple times on the same subject.
 * Whether the metadata is compile-time only, or if its data should persist as rtti.
 * Whether the metadata is restricted to a specific platform (or list of platforms).
 * Whether the metadata requires another metadata to function.

While subject to change, this proposal recommends adding additional metadata to be used
in combination with `@:metadata` to configure these options:
```haxe
/**
	Use on a metadata function. That meta will only exist at
	compile-time (will not generate any rtti data).

	Unresolved question: Should this force this meta to
	be prefixed with a colon? 
**/
@:metadata
@metadataCompileOnly
@metadataRequire(@:metadata _)
function metadataCompileOnly(): haxe.macro.Expr.Function;

/**
	Use on a metadata function.

	Generates an error if the metadata function is
	used on a subject without a meta with the name `other`.

	For example: `@:metadataRequire(Meta.metadata)` only allows
	this meta to be used in combination with `@:metadata`.
**/
@:metadata
@metadataCompileOnly
@metadataRequire(@:metadata _)
function metadataRequire(...other: haxe.macro.Expr.MetadataEntry): haxe.macro.Expr.Function;

/**
	Register a function as a typed meta.

	`allowMultiple` defines whether this metadata can be 
	used multiple times on the same subject.
**/
@:metadata
@metadataCompileOnly
@metadataRequire(@:metadata _)
function metadataAllowMulti(): haxe.macro.Expr.Function;

/**
	Use on a metadata function.

	Restricts the meta to only be used on specific platforms.

	If untyped metadata throw an error, this is unnecessary, as one can
	use conditional compilation to only define this metadata if a target's
	define is defined.
**/
@:metadata
@metadataCompileOnly
@metadataRequire(@:metadata _)
function metadataPlatforms(...platformName: String): haxe.macro.Expr.Function;
```

&nbsp;
&nbsp;

## Allowed Argument Types

The following is the full list of allowed argument types for metadata.

| Type | Expression Must Match | Decorator Argument Value | MetadataEntryTools Getter | Description |
| --- | --- | --- | --- | --- |
| `Bool` | `EConst(CIdent("true" \| "false"))` | `v == "true"` | `getBool` | Allows either `true` or `false`. |
| `Int` | `EConst(CInt(v))` | `Std.parseInt(v)` | `getInt` | Allows an integer literal. |
| `Float` | `EConst(CFloat(v))` or `EConst(CInt(v))` | `Std.parseFloat(v)` | `getFloat` | Allows an float literal. |
| `String` | `EConst(CString(v, DoubleQuotes))` | `v` | `getString` | Allows a string literal. Let there be unique error message if `SingleQuotes` is used. |
| `EReg` | `EConst(CRegexp(s, opt))` | `new EReg(s, opt)` | `getRegex` | Allows a regular expression literal. |
| `haxe.macro.Expr.Var` | `EVars([v])` | `v` | `getVarDecl` | Allows variable declaration expression. |
| `haxe.macro.Expr` | `e` | `e` | `getExpr` | Allows any expression. The expression object is passed directly. |
| `haxe.macro.Expr.TypePath` | `EConst(CIdent(_))` or `EField(_, _)` | ??? | `getTypePath` | Allows a type path. The expression will be converted to a `TypePath` manually by the Haxe compiler. Furthermore, it's only valid if the type path follows Haxe package/module naming rules (packages must be lowercase, module and sub names must start with uppercase). |
| `haxe.macro.Expr.FunctionPath` | `EConst(CIdent(_))` or `EField(_, _)` | ??? | `getFunctionPath` | Same as `TypePath`, but when converting/validating from an expression, this allows the final identifier to start with a lowercase letter. |
| `haxe.macro.Expr.ComplexType` | `ECheckType({ expr: EConst(EIdent("\_")) }, complexType)` | `complexType` | `getComplexType` | Allows any type. Must format as `_ : Type` to comply with expression parsing. |
| `haxe.macro.Expr.MetadataEntry` | `EMeta(metaEntry, { expr: EConst(EIdent("\_")) })` | `metaEntry` | `getMetadataEntry` | Allows any metadata. Must format as `@:meta _` to comply with expression parsing. |

&nbsp;
&nbsp;

## Allowed Return Types

The following is the full list of allowed return types for metadata.

| Type | DecoratorSubjectType Case | Description |
| --- | --- | --- |
| `Any` | N/A | The metadata can be used anywhere. |
| `haxe.macro.Expr` | `DExpression(e: Expr)` | The metadata can only be used on an expression. |
| `haxe.macro.Expr.TypeDefinition` | `DTypeDefinition(td: TypeDefinition)` | The metadata can only be used on type definitions. |
| `haxe.macro.Expr.Field` | `DField(f: Field)` | The metadata can only be used on class fields. |
| `haxe.macro.Expr.TypeParamDecl` | `DTypeParam(tp: TypeParamDecl)` | The metadata can only be used on type parameters. |

&nbsp;
&nbsp;

## Decorators

A typed metadata that has code in its function body is called a Decorator.
The code of a decorator is run for every entry of the typed metadata.

### Context.getDecoratorSubject()

To retrieve information about the subject of the decorator, `Context.getDecoratorSubject` is
a new `Context` function that may be used.

```haxe
class Context {
	// ...
	public static function getDecoratorSubject(): DecoratorSubject { ... }
}
```

`DecoratorSubject` is a new typedef from the `Context` module containing the entry that triggered
the metadata function call and the "target" it's being used on.

`DecoratorSubjectTarget` is a new enum from the `Context` module containing all the possible
metadata targets and their equivalent untyped data structure.

```haxe
import haxe.macro.Expr;

typedef DecoratorSubject = {
	entry: MetadataEntry,
	type: DecoratorSubjectTarget
}

// Prefix with "D" to prevent conflicts with `haxe.macro.` classes?
enum DecoratorSubjectTarget {
	DExpression(e: Expr);
	DTypeDefinition(td: TypeDefinition);
	DField(f: Field);
	DTypeParam(tp: TypeParamDecl);
}
```

### Custom Decorator Validator

Decorators do not need to return a value. If `null` is returned, the decorator will not affect
its subject. However, this can be helpful as this allows developers to write their own logic
for whether a metadata is being used correctly.

If one's metadata should only be used on a SPECIFIC type of expression or a SPECIFIC type of field,
this is where that can be enforced.

Note: when a throw statement is used in a metadata function, it should generate a compiler error
at the position of the metadata entry that triggered the function.
```haxe
// Only works on property fields
@:metadata function propMeta(): haxe.macro.Expr.Field {
	switch(Context.getDecoratorSubject().type) {
		case DField(f): {
			switch(f.kind) {
				case FProp(_, _, _, _): { /* don't do anything */ }
				case _: throw "This metadata should only be used on properties.";
			}
		}
		case _: throw "Impossible";
	}

	return null;
}
```

### Subject-Modifying Decorator

If a decorator's function returns an instance of the "target" subject, that instance
will replace the decorator's subject in the compiler.

For example, this decorator replaces its expression subject with a `0`.

```haxe
@:metadata function makeZero(): haxe.macro.Expr {
	return macro 0;
}

// ---

trace(@makeZero "Hello!"); // Main.hx:1: 0
```

```haxe
@:metadata function changeName(n: String): haxe.macro.Expr.TypeDefinition {
	return switch(Context.getDecoratorSubject().type) {
		case DTypeDefinition(td): {
			td.name = n;
			td;
		}
		case _: throw "Impossible";
	}
}

// ---

@changeName("YourClass")
class MyClass {
	public function new() {}
}

function main() {
	final c = new YourClass();
}
```

A metadata that works on any subject can be smart and perform different actions based
on the subject type.
```haxe
/**
	Adds a meta to any subject.
**/
@:metadata function markWithMeta(name: String): Any {
	return switch(Context.getDecoratorSubject().type) {
		case DExpression(e) {
			{
				expr: TMeta({ name: name, pos: e.pos }, e),
				pos: e.pos
			};
		}
		case DTypeDefinition(td): {
			if(td.meta == null) td.meta = [];
			td.meta.push({ name: name, pos: td.pos });
			td;
		}
		case DField(f): {
			if(f.meta == null) f.meta = [];
			f.meta.push({ name: name, pos: f.pos });
			f;
		}
		case DTypeDefinition(tp): {
			if(tp.meta == null) tp.meta = [];
			tp.meta.push({ name: name, pos: Context.makePosition({min: 0, max: 0, file: ""}) });
			tp;
		}
	}
}
```

&nbsp;
&nbsp;

# Impact on existing code

There will only be an impact on existing code if untyped metadata are decided to generate
errors.

Otherwise, the API additions should not cause any breaking changes and there should be
no impact on existing code.

&nbsp;
&nbsp;

# Drawbacks

There might be of a performance penalty since all metadata must be type checked.

&nbsp;
&nbsp;

# Alternatives

Metadata can be typed checked manually, but requires a lot of unnecessary boilerplate. See [Motivation](0000-typed-metadata.md#motivation). 

Decorators on expressions, fields, and variables can be replicated using global `@:build` macros,
which are significantly slower and require writing boilerplate for checking all expressions/fields.

There is currently no alternatives for decorators on type definitions.

&nbsp;
&nbsp;

# Unresolved questions

Should untyped metadata throw an error? Maybe a warning? Maybe errors can be enabled/disabled with
a define?

How should colons be handled? If the current built-in Haxe metadata is going to be typed, there should
probably be a way to set a metadata to use a colon to ensure compatibility. However, it might be prefered
to encourage/enforce that users are only make typed metadata without a colon? For the time being,
`@metadataCompileOnly` answers this question by requiring a colon but not generating rtti.

Should `MetadataEntry`s `field` field be `Ref<haxe.macro.Type.ClassField>` or `haxe.macro.Expr.Field`?
