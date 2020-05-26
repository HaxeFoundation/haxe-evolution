# Local Variable Metadata Syntax

- Proposal: [HXP-NNNN](0000-local-var-metadata.md)
- Author: [Peter Achberger](https://github.com/antriel)

## Introduction

Haxe allows adding metadata on declarations, fields, and expressions. While we can add metadata to local variable declaration expression, we currently cannot add it to the variable itself.

## Motivation

While we can inspect expression metadata in build macros, we cannot do so in expression macro for expressions of the method, that the macro is called from.
This can be a limitation for expression macros that might want to use the local variables based on some metadata (e.g. dependency injection).

Currently the only solution, that I know of, is moving the metadata along with variable names up to the method, as field metadata, making it readable by the expression macro.
This is less obvious, more prone to errors/typos due to variable name duplication, and more difficult to manage.

See also https://github.com/HaxeFoundation/haxe/issues/9468.

## Detailed design

I propose a new syntax for local variables:

```haxe
var @:meta foo:Bar;
```

`haxe.macro.Type.TVar` returned from `haxe.macro.Context.getLocalTVars()` already has `meta` property that could be filled with this new syntax. Currently it's always empty, because normal syntax of `@:meta var foo:Bar;` adds the metadata on the expression, not the variable declaration.

This syntax shall work regardless of whether the variable has an explicit type, and/or initialization, and shall work with the comma syntax. All those shall be therefore valid:

```haxe
var @:meta foo;
var @:meta foo = 'bar';
var @:meta foo:String;
var @:meta foo, bar:Bool, @:meta c:Int = 0;
```

## Impact on existing code

It's a new syntax, so should be none.

## Drawbacks

The only issue is that the new syntax isn't exactly obvious and goes against the usual one that works for member fields. That means e.g. copying member variables into local variables would require more manual changes to keep the same functionality (assuming the expression macro uses both fields and local variables).

## Alternatives

We could also make the current syntax `@:meta var foo:Bar;` duplicate the metadata from the expression to the `TVar`. That would allow us to keep the same syntax, but it might have consequences/issues that I don't see.

## Unresolved questions

None.
