# Typed Metadata

* Proposal: [HXP-0000](0000-typed-metadata.md)
* Author: [Robert Borghese](https://github.com/RobertBorghese)

&nbsp;
&nbsp;

# Introduction

A typing system for Haxe metadata that can validate its arguments and optionally provide compile-time transformations.

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
	public function printPlus3(a: Null<Int>) {
		@ifSome(a) {
			trace(a + 3);
		}
	}
}
```

&nbsp;
&nbsp;

# Motivation

Sometimes metadata can be a little tedious to work with.

<ins>Making</ins><br/>
When writing code using Haxe metadata, it's simple to check for
a specific metadata's name; however, the arguments are a nightmare.
A lot of boilerplate needs to be written to:
 * check if an argument exists
 * check if it's the desired type
 * convert from `Expr` to a usable data type

<ins>Using</ins><br/>
On the other hand, using someone's Haxe code that processes
metadata can become troublesome. There is no guarentee the metadata
is documented properly, and there is no scoping control to prevent
naming conflicts.

- - -

Typing metadata using function declarations provides a better format
for finding, documenting, and error checking Haxe metadata.

&nbsp;
&nbsp;

# Detailed design

There's a lot to cover here. A table has been provided for your convenience: 

| Topic | Description |
| --- | --- |
| [Basic Rules](0000-typed-metadata.md#basic-rules)                       | The basic syntax and rules for a metadata "function" declaration. |
| [@:metadata Arguments](0000-typed-metadata.md#metadata-arguments)       | The properies to configure @:metadata. |
| [Haxe API Changes](0000-typed-metadata.md#haxe-api-changes)             | The changes to the Haxe API required. |
| [Allowed Argument Types](0000-typed-metadata.md#allowed-argument-types) | List of argument types allowed for a typed metadata. |
| [Allowed Return Types](0000-typed-metadata.md#allowed-return-types)     | List of return types allowed for a typed metadata. |
| [Decorators](0000-typed-metadata.md#decorators)                         | The design of metadata that runs code from its function body. |

&nbsp;
&nbsp;

## Basic Rules

A metadata can be declared using a function declaration with the `@:metadata` meta.

Metadata functions are permitted to lack an implementation (similar to `extern` functions). Typed metadata with function code is still allowed and will be covered later (see [Decorators](0000-typed-metadata.md#decorators)).
```haxe
@:metadata function myMeta(): Any;
```

&nbsp;

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

&nbsp;

### Metadata Scoping/Importing

Typed metadata can be used on anything that allows metadata on it currently.
However, it follows the same scoping rules as Haxe functions. Meaning it must
use its full path or be imported:
```haxe
@mypack.MyModule.myMeta
function doThing() { ... }

// OR

import mypack.MyModule; 

// static function: @MyModule.myMeta
// module level:    @myMeta 

@myMeta
function doThing() { ... }
```

&nbsp;

### Untyped Metadata

If the Haxe compiler encounters a metadata entry it cannot type, its behavior is currently an [Unresolved Question](0000-typed-metadata.md#unresolved-question). 

For the time being, this proposal suggests printing an error for each metadata entry that could not be typed (no metadata function could be found for its name/path) ONLY IF within a module that meets the following conditions:
 * A `@:metadata` function is declared in that module.
 * A `@:metadata` function is imported.
 * A module with a `@:metadata` function is imported (including wildcard imports).
 * At least one metadata in the module has been successfully typed (this counts even if its arguments do not pass typing).

This ensures any user that intends to use typed metadata will receive proper typing. A define can be used to enforce metadata typing on all code used in a Haxe project: `-D strict-meta-typing`

To use an untyped metadata in a "typed metadata" context, the `@:untypedMeta` metadata should be used:
```haxe
@:untypedMeta(something)
@:untypedMeta(another(1, "test"))
class MyClass {}
```

&nbsp;

### Metadata Target

The return type of the metadata function declaration dictates where it's allowed to be used.

The `Any` type denotes a metadata can be used anywhere. The `haxe.macro.Expr` restricts a metadata's usage to expressions. A full list of allowed return types can be found at [Allowed Return Types](0000-typed-metadata.md#allowed-return-types). Any return type besides those are not allowed and should result in an error.

```haxe
// use this anywhere
@:metadata function anyMeta(): Any;

// only allowed on expressions
@:metadata function exprMeta(): haxe.macro.Expr;

// error: Type `Int` is not valid type for metadata function.
@:metadata function intMeta(): Int;
```

&nbsp;

### Basic Meta Arguments

Arguments can be added to the metadata functions. Like with return types, there are only certain types allowed. A full list can be found at [Allowed Argument Types](0000-typed-metadata.md#allowed-argument-types).

Outside the restriction of certain types, arguments should work exactly the same as they do on normal functions. This includes support for: optional arguments, default arguments, and rest arguments.
```haxe
@:metadata function oneNum(num: Int): Any;
@oneNum(123) function doThing() { ... }

// default args
@:metadata function maybeNum(num: Int = 0): Any;
@maybeNum function doThing() { ... }
@maybeNum(123) function doThing2() { ... }

// optional args
@:metadata function numAndStr(?num: Int, str: String): Any;
@numAndStr(123, "test") function doThing() { ... }
@numAndStr("test") function doThing2() { ... }

// rest args
@:metadata function numRest(...num: Int): Any;
@numRest function doThing() { ... }
@numRest(1) function doThing2() { ... }
@numRest(1, 2, 3) function doThing3() { ... }

// error: Type `haxe.Exception` is not valid argument type for metadata function.
@:metadata function invalidType(o: haxe.Exception): Any;
```

&nbsp;
&nbsp;

## @:metadata Arguments

There needs to be a way for metadata functions to configure a couple options:
 * Can it be used multiple times on the same subject?
 * Is it compile-time only (uses `@:`)? Or should it exist as rtti.
 * Is it restricted to one or more platforms?
 * Does it require another metadata to function?

To resolve these, the `@:metadata` metadata provides a couple options that can be configured. The declaration for the `@:metadata` metadata would look something like this:
```haxe
@:metadata function metadata(?options: {
    ?allowMultiple: Bool,
    ?compileTime: Bool,
    ?platforms: Array<String>
}): haxe.macro.Expr.Function;
```

| Argument Name | Default Value | Description |
| --- | --- | --- |
| allowMultiple | `false` | If `true`, this metadata can be used on the same subject multiple times. |
| compileTime | `false` | If `true`, this metadata must be prefixed with a colon and does not generate rtti. |
| platforms | `[]` | If this Array contains at least one entry, this metadata can only be used on the platforms named. |

These options are optional, but they can be overriden if needed:
```haxe
@:metadata({ allowMultiple: true })
function author(name: String): Any;

@:metadata({ compileTime: true })
function tempData(): Any;

@:metadata({ allowMultiple: true, platforms: ["java", "cs"] })
function nativeMeta(m: Expr): Any;

// ---

@author("Me")
@author("You")
@:tempData
function myFunc() {
}
```

&nbsp;
&nbsp;

## Haxe API Changes

A new optional field should be added to `haxe.macro.Expr.MetadataEntry`.

If this metadata entry is typed, then `field` will contain a reference to the `ClassField` of the metadata function.
```haxe
// Unresolved question
// Would it be possible to use Ref<Type.ClassField> instead?
var ?field: Expr.Field;
```

&nbsp;

### Reading Arguments

There needs to be a mechanism for reading metadata arguments. To provide this, a new field `typedMeta: StringMap<Dynamic>` should be added to:
 * `haxe.macro.Expr.TypeDefinition`
 * `haxe.macro.Expr.Field`
 * `haxe.macro.Expr.TypeParamDecl`

The entires correlate directly to the full path of the metadata used on the subject. So to access the content of an `@Meta.date` metadata, `_.typedMeta.get("mypack.Meta.date")` must be used. This is to prevent naming conflicts. There may be multiple metadata of the same name in different modules.

The `Dynamic` value contains fields with the same name as the arguments of the typed metadata. These fields store the values passed to the metadata entry. How these values are converted can be viewed in [Allowed Argument Types](0000-typed-metadata.md#allowed-argument-types).

If the metadata has `allowMultiple` enabled, the `Dynamic` value will ALWAYS be an Array, even if only one instance of the metadata is used.
```haxe
// MyModule.hx
package mypack;

@:metadata({ allowMultiple: true })
function author(name: String): TypeDefinition;

class Meta {
   @:metadata
   public static function date(month: Int, day: Int): TypeDefinition;
}
class AnotherMeta {
   @:metadata
   public static function date(dateString: String): TypeDefinition;
}

@author("Something")
@Meta.date(11, 15)
@AnotherMeta.date("November 15, 2004")
class MyClass {}
```

```haxe
// ---
// in some compile-time function
// var td: TypeDefinition;
final authorNames: Null<Array<String>> = td.typedMeta.get("mypack.MyModule.author")?.map(meta -> meta.name);

final dateMonth: Int = td.typedMeta.get("mypack.MyModule.Meta.date")?.month;
```

&nbsp;

### Context.typeMetadata

A new static function should be added to `haxe.macro.Context`:
```haxe
class Context {
    // ...
    public static function typeMetadata(meta: haxe.macro.Expr.Metadata): StringMap<Dynamic> { ... }
```

This is a function that will generate an object like the `typedMeta: StringMap<Dynamic>` field described in the previous section. This would be helpful for extracting typed metadata data found in untyped `EMeta` expressions.

&nbsp;

### Function Type Struct

The following anonymous structure should be added to the `haxe/macro/Expr.hx` module:
```haxe
typedef FunctionPath = {
	> TypePath,
	functionName: String;
};
```

This is a structure for storing type paths to functions. It is used as an argument type for metadata. Long story short, it allows for type paths that end with a lowercase identifier (`myFunc`, `Module.Sub.myFunc`).

Technically, function path data _could_ be stored in `TypePath`, but that's not preferable.

&nbsp;
&nbsp;

## Allowed Argument Types

The following is the full list of allowed argument types for metadata.

| Type | Expression Must Match | Decorator Argument Value | Description |
| --- | --- | --- | --- |
| `Bool` | `EConst(CIdent("true" \| "false"))` | `v == "true"` | Allows either `true` or `false`. |
| `Int` | `EConst(CInt(v))` | `Std.parseInt(v)` | Allows an integer literal. |
| `Float` | `EConst(CFloat(v))` or `EConst(CInt(v))` | `Std.parseFloat(v)` | Allows an float literal. |
| `String` | `EConst(CString(v, DoubleQuotes))` | `v` | Allows a string literal. Let there be unique error message if `SingleQuotes` is used. |
| `EReg` | `EConst(CRegexp(s, opt))` | `new EReg(s, opt)` | Allows a regular expression literal. |
| `haxe.macro.Expr.Var` | `EVars([v])` | `v` | Allows variable declaration expression. |
| `haxe.macro.Expr` | `e` | `e` | Allows any expression. The expression object is passed directly. |
| `Array<TYPE>` | `EArrayDecl(_)` | ??? | Allows array declarations. `TYPE` should be a from this list. Requires some internal logic to convert `Array<Expr>` into the `TYPE`. |
| `{ name: TYPE, ... }` | `EObjectDecl(_)` | ??? | Allows object declarations. All types used should be from this list. Requires some internal logic to convert `Array<ObjectField>` into a `Dynamic` with the fields. |
| `haxe.macro.Expr.TypePath` | `EConst(CIdent(_))` or `EField(_, _)` | ??? | Allows a type path. The expression will be converted to a `TypePath` manually by the Haxe compiler. Furthermore, it's only valid if the type path follows Haxe package/module naming rules (packages must be lowercase, module and sub names must start with uppercase). |
| `haxe.macro.Expr.FunctionPath` | `EConst(CIdent(_))` or `EField(_, _)` | ??? | Same as `TypePath`, but when converting/validating from an expression, this allows the final identifier to start with a lowercase letter. |
| `haxe.macro.Expr.ComplexType` | `ECheckType({ expr: EConst(EIdent("\_")) }, complexType)` | `complexType` | Allows any type. Must format as `_ : Type` to comply with expression parsing. |
| `haxe.macro.Expr.MetadataEntry` | `EMeta(metaEntry, { expr: EConst(EIdent("\_")) })` | `metaEntry` | Allows any metadata. Must format as `@:meta _` to comply with expression parsing. |

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

A typed metadata that has code in its function body is called a "decorator". A decorator's code is run for every entry of the typed metadata.

&nbsp;

### Context.getDecoratorSubject()

To retrieve information about the subject of the decorator, `Context.getDecoratorSubject` is a new `Context` function that may be used.

```haxe
class Context {
	// ...
	public static function getDecoratorSubject(): DecoratorSubject { ... }
}
```

`DecoratorSubject` is a new typedef from the `Context` module containing the `MetadataEntry` that triggered the call and the target.

```haxe
typedef DecoratorSubject = {
	entry: MetadataEntry,
	type: DecoratorSubjectTarget
}
```

`DecoratorSubjectTarget` is a new enum containing all the possible metadata targets and their "Expr" data structure.

```haxe
import haxe.macro.Expr;

// Prefix with "D" to prevent conflicts with `haxe.macro.` classes?
enum DecoratorSubjectTarget {
	DExpression(e: Expr);
	DTypeDefinition(td: TypeDefinition);
	DField(f: Field);
	DTypeParam(tp: TypeParamDecl);
}
```

&nbsp;

### Custom Decorator Validator

Decorators do not need to return a value. If `null` is returned, the decorator will not affect its subject. Developers can use this to write their own logic for ensuring their metadata is used correctly.

If one's metadata should only be used on a SPECIFIC type of expression or a SPECIFIC type of field, this is where that can be enforced.
```haxe
// Only works on property fields
@:metadata function propMeta(): haxe.macro.Expr.Field {
	switch(Context.getDecoratorSubject().type) {
		case DField(f): {
			switch(f.kind) {
				// Do something with property
				case FProp(_, _, _, _): {  }
				
				// Let the user know the metadata was used incorrectly!
				case _: Context.error("This metadata should only be used on properties.", Context.getDecoratorSubject().entry.pos);
			}
		}
		case _: throw "Impossible";
	}

	return null;
}
```

&nbsp;

### Subject-Modifying Decorator

If a decorator's function returns an non-null instance of its return type, that instance will replace the decorator's subject at compile-time.

```haxe
@:metadata({ compileTime: true })
function makeZero(): haxe.macro.Expr {
	return macro 0;
}

// ---

trace(@:makeZero "Hello!"); // Main.hx:1: 0
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

A metadata that works on any subject can be smart and perform different actions based on the type of subject it was used on.
```haxe
/**
	Adds a meta to any subject.
**/
@:metadata function markWithMeta(name: String): Any {
	return switch(Context.getDecoratorSubject().type) {
		case DExpression(e): {
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

There will only be an impact on existing code if untyped metadata generate errors.

Otherwise, the API additions do not cause any breaking changes, and there should be no impact on existing code.

&nbsp;
&nbsp;

# Drawbacks

There might be of a performance penalty since all metadata have to look up if they're typed?

&nbsp;
&nbsp;

# Alternatives

Metadata can be typed checked manually, but requires a lot of unnecessary boilerplate. See [Motivation](0000-typed-metadata.md#motivation). 

Decorators on expressions, fields, and variables can be replicated using `@:build` macros, which are significantly slower and require writing boilerplate for checking all expressions/fields.

There is currently no alternatives for decorators on type definitions.

&nbsp;
&nbsp;

# Unresolved questions

Should a new metadata syntax be used: `@.myMeta`? This would ensure all new metadata could be typed properly.

If no `@.` syntax, should untyped metadata throw an error? While it would be a major breaking change, it would be nice restrict metadata usage using conditional compilation (wrap with `#if js` for example) instead of using something like `@:metadataPlatform`. Maybe it could be a warning? Maybe errors can default to on, but turn off with a define (or vise versa)?

How should colons be handled? If the current built-in Haxe metadata is going to be typed, there should probably be a way to set a metadata to use a colon to ensure compatibility. However, it might be prefered to encourage/enforce that users are only make typed metadata without a colon? For the time being, `@metadataCompileOnly` answers this question by requiring a colon but not generating rtti.

Should `MetadataEntry`s `field` field be `Ref<haxe.macro.Type.ClassField>` or `haxe.macro.Expr.Field`? Would it be possible to type the field that early?
