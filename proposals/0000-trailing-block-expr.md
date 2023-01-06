# Trailing Block Expressions

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Robert Borghese](https://github.com/RobertBorghese)

## Introduction

Allow block expressions to be appended to macro calls as final argument.
```haxe
macro function ifNotNull(expr: Expr, trail: TrailingExpr): Expr {
	return macro {
		final val = $expr;
		if(val != null) {
			$trail;
		} else null;
	}
}

// ---

ifNotNull(generateData()) {
	trace(val);
}
```

## Motivation

This feature has two driving motivations: clarify, optimize, and simplify a somewhat common pattern in Haxe metaprogramming, and open oppurtunities for better Haxe integrated DSLs.

### Simplify

There are currently two ways to replicate this functionality currently. 

First is by placing the block within the macro call: `myCall({ ... })`. This syntax is identical to a normal function with a block expression being used as a value, so it could be confusing. Not to mention it's uglier and clunkier to write. The proposed new syntax clarifies the expression is being processed through a macro, as it cannot be used elsewhere.

The second method is with metadata: `@myMeta { ... }`. It looks nearly identical to the proposed syntax, but modifying the attached expression requires iterating through the entire project's AST to find the metadata using global `@:build`. Furthermore, metadata is untyped, so there is no namespace/import control, and no way to easily find the source reading the code. Meanwhile, a macro function is imported, typed, and can be sourced easily with IDE tools, so it is an objective improvement.

### DSL

In addition to the technical capabilities provided to macros, allowing the Haxe parser to accept trailing block expressions becomes a feature within itself. While not the conclusive answer to all DSL support, it provides a fast, easy, Haxe-ified, type-safe alternative.

In fact, a decent number of newer languages' support for "DSL" seemingly boils down to allowing the parser to accept trailing blocks. See the "Macros to implement DSLs" section on this [official Nimlang blog](https://nim-lang.org/blog/2021/11/15/zen-of-nim.html), or check out this [Kotlin tutorial](https://kotlinlang.org/docs/type-safe-builders.html#scope-control-dslmarker) and [Kotlin article](https://medium.com/kotlin-and-kotlin-for-android/kotlin-dsl-coding-a-dsl-6-ee355be81106).

```haxe
// how an html builder in Haxe could look
return html {
  head {
    title { "My Page"; }
  }
  body {
    div {
      p(style="mystyle") {
        "Hello World";
      }
    
      button(onclick=Statics.whenButtonPress) {
        "Click me";
      }
    }
  }
}
```

<br />

## Detailed design

### Structure Changes

A new `ExprDef` case should be added.
```haxe
ETrailingBlock(e: Expr, blockExpr: Expr)
```

And a new typedef named `TrailingExpr` should be added to `haxe.macro.Expr`. This is so a macro function can choose to accept a trailing block vs normal expression, while also ensuring `TrailingExpr`s are compatible with macro reification. 
```haxe
typedef TrailingExpr = Expr;
```

<br />

### Typing Rules

A trailing block is a metaprogramming feature. All instances of `ETrailingBlock` should be converted prior to the typing phase either using macro functions or `@:build` macros. If the typer encounters `ETrailingBlock`, an error is thrown (though, other alternatives to how to handle this are listed below).

As a result, there is no `TypedExpr` equivalent for `ETrailingBlock`.

<br />

### Declaration Rules

For a macro function to accept a trailing expression, the last argument must be `TrailingExpr`.
```haxe
macro function useTrail(num: Int, e: Expr, trail: TrailingExpr); // valid
macro function useTrail(trail: TrailingExpr, num: Int); // error: TrailingExpr must be last argument
```

`TrailingExpr` is incompatible with the new rest argument syntax (`...`). However, a macro function that uses both rest arguments and a trailing block expression can be created by making the second to last argument an `Array<Expr>`, and the final argument `TrailingExpr`.
```haxe
macro function f(trail: TrailingExpr, ...e: Expr); // error: TrailingExpr must be last argument
macro function f(trail: TrailingExpr, args: Array<Expr>); // error: TrailingExpr must be last argument

macro function f(e: Expr, args: Array<Expr>); // macro function with Expr rest arguments
macro function f(args: Array<Expr>, e: Expr); // macro function that does NOT accept rest arguments
macro function f(args: Array<Expr>, trail: TrailingExpr); // macro function that acceps rest arguments AND trailing block
```

`TrailingExpr` is only allowed in macro functions.
```haxe
// error: haxe.macro.Expr.TrailingExpr only allowed in macro functions. Use haxe.macro.Expr instead.
function f(trail: TrailingExpr);
```

<br />

### Syntax Rules

A trailing block can be placed after any function call. Simply place a block expression directly after the closing parenthesis of the call.
```haxe
myCall(12, "string") {
  // place block content here
}
```

If the call expression does not have any arguments (besides the `TrailingExpr`), the parenthesis can be omitted.
```haxe
onlyOneArg {
  // place block content here
}
```

<br />

## Impact on existing code

A new case is added to `haxe.macro.ExprDef`, so will break some macro code.

The new syntax itself should not cause any issues with existing code, however.

## Drawbacks

Maybe certain syntax errors might not trigger the same? But the restrictive nature of the new syntax should ensure it's not an issue.

## Alternatives

See Motivation.

## Unresolved questions

Perhaps instead of using `TrailingExpr`, the trailing block can be used on any macro with an `Expr` as the final argument?

Maybe the trailing expression can be used in normal functions, but when passing a function for the last argument similar to Kotlin?
