# Type parameter variance of enum

* Proposal: HXP-NNNN
* Author: [shohei909](https://github.com/shohei909)

## Introduction

Supports type parameter covariance and contravariance of enum.

## Motivation

A common case is to assign an `Option` to another `Option`.

```haxe
var option:Option<Float> = Option.Some(1);
```

Currently, following compile errors occur.

```
haxe.ds.Option<Int> should be haxe.ds.Option<Float>
Type parameters are invariant
Int should be Float
```

There are two methods to avoid this. The first is `cast`.

```haxe
var option:Option<Float> = cast Option.Some(1);
```

However, `cast` is not safe.

The second is re-instantiating the `Option`.

```haxe
class OptionTools {
    public static function upcast<T, U:T>(option:Option<U>):Option<T> {
        return switch (option) {
            case Option.Some(v): Option.Some((v:T));
            case Option.None:    Option.None;
        }
    }
}
```

```haxe
var option:Option<Float> = OptionTools.upcast(Option.Some(1));
```

That is safe, but introduces runtime overhead.

If covariance is supported, `Option<Int>` can be `Option<Float>`.

## Detailed design

In terms of variance, the arguments of each constructor of enum are same as the get-only properties. 

### Contravariance

The following Sample enum can be contravariant.

```haxe
enum Sample<T> {
    A(T->Void);
}
```

```haxe
var sample:Sample<Int> = Sample(function (f:Float) {});
```

### Compiler output

In dynamic targets, the compiler output is the same as when `cast`. 

In the current static targets (Flash, C++, C#, Java), enum type parameter seems to be disappeared from output, so it should be same.

### GADT

Variance of GADT(generalized algebraic data type) can break exisiting code.

```haxe
enum Gadt<T> {
    I(i:Int):Gadt<Int>;
    F(f:Float):Gadt<Float>;
}
```

`Gadt<Float>` doesn't have `I` constructor. The following code causes no compile error.

```haxe
switch (Gadt.F(0.0)) {
    case Gadt.F(_):
}
```

If `Gadt<T>` is covariant, `Gadt<Float>` should have `I` constructor.

Therefore, I think that GADT should be invariant at least by default.

## Impact on existing code

No impact on existing code.

## Alternatives

Alternatives are described in [Motivation](#motivation) section.

## Drawbacks

This may make some feature additions difficult, such as [Enum member method/variables](https://github.com/HaxeFoundation/haxe-evolution/issues/10).
