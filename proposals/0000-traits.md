# Traits

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Haxe Developer](https://github.com/haxedev)

## Introduction

Introduces a new top-level declaration type to haxe that acts like an interface, but allows default methods to be added. 
It also allows one to implement these traits anywhere on any class.

## Motivation

This is motivated by language systems Rust and Java. These languages support 
default implementations in traits and interfaces respectively. This, however is just a small portion of why
it should be added. Taking heavily from Rust, this also allows one to define new methods 
on a class anywhere. A great example from haskell is binary serialization - the binary lib automates 
much of serialization, and helped me personally write seamless bytecode
generators. Having something like this for haxe could make many things much easier.

## Detailed design

Traits syntax is the same as interfaces, except the interface keyword is swapped for a new "trait" keyword,
and that methods can have bodys, and variables can have definitions. 


The major difference between traits and interfaces is default methods. Instead of a semicolon at the end of the type definition,
the programmer may specify a method body. As the type is important to implementors wanting to override, 
type inference for the return type and arguments is not allowed. 

Variables may also have definitions; When implementing a trait with default variables, you may specify "override" on 
var definitions to specify a new default definition. You may not override a var without specifying a default definition.

Override variables would look like this: 
```hx
trait MyTrait {
  var someVar = "foo";
}

class MyClass implements MyTrait {
  override var someVar = "bar"; 
}

class BadClass implements MyTrait {
  // not allowed 
  override var someVar; // you need to specify a definition. 
}
```

As you can see, implementing has the same syntax as interfaces. If methods or variables from different interfaces
have the same name, then you may not implement them both directly on the class. 

You may override functions with default implementations, in the same way you can override interface methods. You
_will_ have access to the original method via super. 

Traits can extend each other like interfaces - you can override default functions with new default implementations. You 
have access to the default method via super. You can extend multiple traits, however if methods or variables have the same
name it will be a compiler error. Interfaces may not extend traits. 


Of course, to truly be inspired by Rust, you must be allowed to implement traits even outside of the class definition. 
Therefore, a new top level declaration will also need to be implemented. 

A syntax of 
```hx
implements Trait for Class
```
will allow one to implement traits anywhere. However, if neither the Trait nor Class was defined in the module, it will
cause a compiler warning about orphan implementations. 


```hx
trait MyTrait {}
// this
class MyClass implements MyTrait {}
// is the same as
class MyClass {}
implements MyTrait for MyClass {}
```

Importing Traits from a module will import all implementations in that module relating to the Trait as well. Implementations
cannot be explicitly imported. 

`implements` syntax will be the same as a class definition, except that you may not define any new fields - you may only define
fields that are in the trait you are implementing. You also can't specify an override or visibility modifier - all functions and variables are 
assumed to be override and public. 

Unlike directly implementing on a class, you _may_ have multiple methods of the same name - the same is still not true with variables.
If two names conflict, you may use fully qualifed syntax. Use the traits name, then field access the method, then call with the
object as the first argument. This looks something like 
```hx
Trait.method(obj, rest...)
```

When compiling to targets, most targets will not support rich traits like this. Traits will therefore be transformed into "static classes" and
interfaces. Static classes are just classes that have all static methods. In languages without concepts of classes, this will be a singleton object.
Interfaces are the same as normal haxe interfaces - it will be like the trait definition but stripped of default methods. 

Implementations will be compiled away, and all methods defined in them will be copied to the original class, and all traits implemented will be added
to the implements list. All methods names will be mangled with the full trait path, with `.` converted to `_`, then the method name after another `_`. 
The trait static methods will then call these mangled names. 

```hx
trait MyTrait {
  function foo():Void {}
}
class MyClass implements MyTrait {}

function main() {
  var klass = new MyClass();
  klass.foo();
}
``` 
will be compiled away to
```hx
interface MyTraitImplementor {
  function __hx_MyTrait_foo():Void; 
}
class MyTrait {
  public static function foo<T:MyTraitImplementor>:(__hx_impl:T):Void {
    __hx_impl.__hx_MyTrait_foo(); 
  }
}
class MyClass implements MyTraitImplementor {
  public override function __hx_MyTrait_foo():Void {
    
  }
}
```
When using traits as arguments, you may only specify it in a generic manner, i.e. `foo<T:MyTrait>(f:T)`. 





## Impact on existing code

Given the scope of this, something will surely break. However, I am not sure what exactly will break. 
## Drawbacks

I am truly uncertain of any drawbacks here; I personally believe that in Rust and Haskell, traits and 
type classes respectively, are some of the best features in those languages. I feel that allowing implementations anywhere 
will really open up the inheritance heirarchy, allowing rich type systems. 
## Alternatives

Static extensions fill some of the niche of traits, however the sheer power of traits and implementations can't be understated. Haskell 
runs on type classes, its version of traits, and most libraries simply introduce new type classes and declare instances for common types. 


## Opening possibilities

The sheer amount of allowed options coming from implementations can, as I stated earlier, really open up the
type system. Haskell and Rust have very rich type systems, in no small part due to traits and implementations. 

Opening this up to any type, this could allow true operator overloading in the future.

This could also make iteration methods and toString a real thing instead of just compiler sillyness. Implementing a "Show" trait would be much more useful.

## Unresolved questions

I am unsure if I covered all of the inheritance possibilites (I likely didn't). Please point out any edge cases I forgot. 

I am not sure how to remove runtime overhead; I would say that copying it is enough, however I'm probably missing something.
