# Intersection Types

* Proposal: [HXP-0004](0004-intersection-types.md)
* Author: [Simn](https://github.com/simn)
* Status: implemented in 4.0.0

## Introduction

Introduce the notion of intersection types using a `Type1 & Type2` syntax.


## Motivation

There are currently two cases in the Haxe language which support some notion of intersection types:

### Structural Extensions

```haxe
typedef Type1 = {
	var field1:String;
}

typedef Type2 = {
	var field2:Int;
}

typedef Type3 = {
	> Type1,
	> Type2,
	var field3:Bool;
}
```

Here, `Type3` is the intersection of types `Type1` and `Type2`. It contains all fields of `Type1` plus all fields of `Type2`.

While the functionality is fine (and would be adopted by this proposal), the syntax is somewhat curious. This would be replaced by the following:

```haxe

// Type1 and Type2 remain the same

typedef Type3 = Type1 & Type2 & {
	var field3:Bool;
}
```

Since the order of types does not matter, this can be formatted in various ways, such as:

```haxe
typedef Type3 = {
	var field3:Bool;
} & Type1 & Type2;

typedef Type3 =
	Type1 &
	Type2 & {
	var field3:Bool;
}
```

### Type parameter constraints

Haxe allows constraining type parameters to multiple types at once:

```haxe
class Class<Param:(Type1, Type2)> { }
```

This reads: If a type is used in place of `Param`, it must unify with `Type1` and it must unify with `Type2`. Logically, this could also be expressed as "it must unify with (`Type1` and `Type2`)".

As before, the functionality is fine, but the syntax is not. In fact, it even introduces [syntactic ambiguities](https://github.com/HaxeFoundation/haxe/issues/7006) which are hard to understand. We propose to use this instead:

```haxe
class Class<Param:Type1 & Type2> { }
```

This also happens to be the syntactic version of "it must unify with (`Type1` and `Type2`)" mentioned above.


## Detailed design

### Parser

After parsing a type-hint, we accept a `&` token and then expect another type-hint. The resulting type-hint would then be a new variant to `complex_type`, `CTIntersection of type_hint * type_hint`. By doing this recursively, we support any number of type-hint operands.

### Typer

When loading a `CTIntersection`, we restrict it like so:

* As long as all operands are structure types or resolve to structure types via typedefs, we create the intersection type like we currently do for `CTExtend`.

* When typing a type parameter constraint, we resolve all operands of `CTIntersection` individually and recursively. This results in a list of types, which is used for constraints checking like the current constraints list is.

* In any other case, we emit an error stating that intersection types cannot be used like that. This may change in the future, but for now the goal is to achieve parity with the current feature set.

It should be noted that the first item in this list takes priority. That is, if we have a type parameter constraint to a structural intersection, we unify once against the intersection, not multiple times against its operands.


## Impact on existing code

Structural extension syntax would be deprecated but still work. However, the current type parameter constraints syntax should be removed, which breaks code using it.

The introduction of a new variant to `complex_type` (`ComplexType`) might require some macros to add additional patterns.

For better backward compatibility, `TypeParamDecl.constraints` can remain an `Array<ComplexType>` which will only ever have 0 or 1 entries.


## Drawbacks

I have to implement it.


## Alternatives

Instead of this rather general approach, the individual problems of structural extensions and type parameter constraints could be looked into:

* The [type parameter constraints syntax conflict](https://github.com/HaxeFoundation/haxe/issues/7006) would have to be addressed, likely breaking it anyway.

* The syntax for structural extensions should be reviewed. It [shouldn't require a `,`](https://github.com/HaxeFoundation/haxe/issues/7036) and should probably support `;` for class field notation as well.

* The result of [unifying against structural extension components](https://github.com/HaxeFoundation/haxe/issues/5225) should be reviewed because the current behavior is not in-line with the overall unification behavior.


## Opening possibilities

Since the goal for now is only feature parity, it doesn't open any new possibilities outside of macros.

In the future, we can support intersection types for interfaces as well. If you have lots of classes which use `implements Interface1 implements Interface2` (and potentially many more), you could instead `typedef Interface3 = Interface1 & Interface2` and use `implements Interface3`. This would work exactly like before, but be easier to write and maintain, especially in interface-heavy code.

## Unresolved questions

None!
