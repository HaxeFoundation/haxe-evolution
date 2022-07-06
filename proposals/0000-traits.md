# Traits

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Haxe Developer](https://github.com/haxedev)

## Introduction

Introduces a new top-level declaration type to haxe that acts like an interface, but allows default methods to be added. 
It also allows one to implement these traits anywhere on any class.

## Motivation

This is motivated by language systems Rust and Java. These languages support 
default implementations in traits and interfaces respectively. Given that default
implementations on interfaces have been rejected, this is the only remaining way 
that they can be implemented in the compiler. Currently, there are macros that allow
partial implementations, however they aren't, in my opinion, as good as a natural implementation, 
like Java or Rust. 

Taking heavily from Rust, this also allows one to define new methods on a class anywhere, and to allow truly generic functions
to exist that actually do useful things. 

## Detailed design

Traits syntax is the same as interfaces, except the interface keyword is swapped for a new "trait" keyword,
and that methods can have bodys, and variables can have definitions. 

The major difference between traits and interfaces is default methods. Instead of a semicolon at the end of the type definition,
the programmer may specify a method body. As the type is important to implementors wanting to override, 
type inference for the return type and arguments is not allowed. 

Variables may also have definitions; When implementing a trait with default variables, you may specify "override" on 
var definitions to specify a new default definition. You may not override a var without specifying a default definition.

Implementing a trait is the same as implementing an interface - use `implements` then the trait name. 
While implementing, methods of the same name may conflict. If they do conflict, then it will be a compiler error. 

You may override functions with default implementations, in the same way you can override interface methods. You
will not have access to the original default method.

Traits can extend each other like interfaces - you can override default functions with new default implementations. You 
have access to the default method via super. You can extend multiple traits, however if methods or variables have the same
name it will be a compiler error. Interfaces may not extend traits.  


Of course, to truly be inspired by Rust, you must be allowed to implement traits even outside of the class definition. 
Therefore, a new top level declaration will also need to be implemented. The `implements` and `for` keywords will be reused. 

A syntax of 
```hx
implements Trait for Class
```
will allow one to implement traits anywhere. However, if neither the Trait nor Class was defined in the module, it will
cause a compiler warning about orphan implementations. 

Importing Traits from a module will import all implementations in that module relating to the Trait as well. Implementations
cannot be explicitly imported. 

`implements` syntax will be the same as a class definition, except that you may not define any new fields - you may only define
fields that are in the trait you are implementing. You also can't specify an override - all functions and variables are 
assumed to be override. 

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

When using traits as arguments, you may only specify it in a generic manner, i.e. `foo<T:MyTrait>(f:T)`. 





## Impact on existing code

Given the scope of this, something will surely break. However, I am not sure what exactly will break.
## Drawbacks

I am truly uncertain of any drawbacks here; I personally believe that in Rust and Haskell, traits and 
type classes respectively, are some of the best features in those languages. I feel that allowing implementations anywhere 
will really open up the inheritance heirarchy, allowing rich type systems. 
## Alternatives

Simply adding default methods to interfaces would fix this, but this has already been rejected as a language feature.

There are also macros that acheive this, however other languages that have macros still have traits (see Rust, Scala)


## Opening possibilities

The sheer amount of allowed options coming from implementations can, as I stated earlier, really open up the
type system. Haskell and Rust have very rich type systems, in no small part due to traits and implementations. 

Opening this up to any type, this could allow true operator overloading in the future. 

## Unresolved questions

I am unsure if I covered all of the inheritance possibilites (I likely didn't). Please point out any edge cases I forgot. 

I am not sure how to remove runtime overhead; I would say that copying it is enough, however I'm probably missing something.
