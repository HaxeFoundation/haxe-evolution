# Complex type expressions

* Proposal: [HXP-NNNN](NNNN-complex-type-expressions.md)
* Author: [Ben Morris](https://github.com/bendmorris)

## Introduction

Currently Haxe can express complex types as expressions or metadata arguments if and only if they parse as another type of expression, such as an identifier or field access. This proposal adds new syntax to express complex types which do not fit this criteria.

## Motivation

There is currently some counter-intuitiveness regarding how types can be used as expressions.

Currently types without parameters can be parsed into either a `EConst(CIdent(...))` or a `EField(...)`, and have a corresponding runtime value:

```haxe
class Test {
    static function main() {
        var a = Test;
        trace(a);
        // Test.hx:4: Class<Test>

        var b = sys.net.Socket;
        trace(b);
        // Test.hx:8: Class<sys.net.Socket>
    }
}
```

The following example does not currently work as they attempt to parse as binary operations instead:

```haxe
class Test {
    static function main() {
        var a = Array<Test>;
        // Test.hx:3: characters 28-29 : Unexpected ;
        // (for some reason I'll get that warning twice)
    }
}
```

These limitations also apply to metadata, so the following also fails to parse:

```haxe
@:myMeta(Array<Test>) public function main() {}
```

However, these can be mitigated by using a typedef:

```haxe
typedef TestArray = Array<Test>;

class Test {
    static function main() {
        var a = TestArray;
        trace(a);
        // Test.hx:6: Class<Array>
    }
}
```

This suggests the only blocker to representing expressions this way is the lack of syntax with which to do so.

Some example use cases for expressing types like this include:

- Constructing the type argument for `Std.is` or `Std.instance`.
- Passing type information as a metadata argument for use by macros.

This would also simplify the cases that already work; rather than checking for `EField` or `EConst` and guessing whether they should be interpreted as types, this syntax allows a type to be expressed with clear intent.

## Detailed design

The new syntax would interpret an expression beginning with `:` as an `EComplexType(ct:ComplexType)`:

```haxe
class Test {
    static function main() {
        var a = :Array<Test>;
        trace(a);
        // Test.hx:3: Class<Array>
    }
}
```

The runtime value of these expressions would work as it already does via typedefs.

This syntax is unambiguous, as there is not currently any other type of expression which begins with a `:`. Notably it does not collide with the ternary syntax; e.g. this: `A ? B : C : D` is a ternary (`A ? B : C`) followed by a complex type (`: D`) which is invalid.

## Impact on existing code

Only in macros. Exhaustiveness checks in macros over `ExprDef` variants may fail once this is introduced.

## Drawbacks

This adds a new expression variant which must be considered in the compiler or in macros.

## Alternatives

Typedefs are an alternative, but having to create a typedef per usage is unfortunate.

An alternative would be to use some other syntax and interpret it as a type, such as:

```haxe
@:optionOne("string.as.Type<Abc>")
@:optionTwo(var _:typed.expr.as.Type<Abc>)
```

These are hacks; having a dedicated syntax to express this would be preferable.

## Opening possibilities

A possible use case for this is to express type signatures in `@:const` metadata (proposed elsewhere.)

## Unresolved questions

None.
