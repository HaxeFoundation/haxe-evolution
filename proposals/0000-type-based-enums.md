# Type Based Enums

* Proposal: [HXP-002](0020-type-based-enums.md)
* Author: [Nicolas Cannasse](https://github.com/ncannasse)
* Status: TBD

## Introduction

Provide optimization for type-based type-safe enum switch

## Motivation

When you want to have a value that can be of two different types (for example ClassA or ClassB, both being unrelated), you can either use:

```
// unsafe variant
abstract AorB(Dynamic) from ClassA from ClassB {
    @:to public function toA() : ClassA { return cast(this,ClassA); }
    @:to public function toB() : ClassB { return cast(this,ClassB); }
}

// type safe variant
enum AorB {
   A( v : ClassA );
   B( v : ClassB );
}
```

The second enum solution is the most type safe but also creates a runtime overhead/wrapper enum constructor.

The idea is to eliminate entirely the overhead by using runtime type testing while keeping the enum syntax and switch.

## Detailed design

You would simply add `@:typeBased` in front of the enum declaration. 
It would then turn this enum into an abstract but on which you can `switch` just like normal enums. In which case it will use `is` operator to test the type instead of the enum matching.

Following example:

```haxe
@:typeBased enum AorB {
   A( v : ClassA );
   B( v : ClassB );
}

function foo( v : AorB ) {
   return switch( v ) {
   case A(a): a.x;
   case B(b): b.get();
   }
}

// would compile to:
function foo( v : AorB_Impl ) {
   return if( v is ClassA ) { var a : ClassA = v; a.x; } else { var b : ClassB = v; b.get(); };
}
```

## Impact on existing code

None. Potential for adding the metadata to optimize existing code

## Drawbacks

Would only work for types than can be tested with "is". Would not allow null.
Would require to test each type one by one (or use divide-and-conquer if they have common types)

## Alternatives

Be able to implement the switch itself in the resulting AST through macros on abstract or some other way

