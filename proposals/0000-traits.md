# Traits

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Haxe Developer](https://github.com/haxedev)

## Introduction

Introduces a new top-level declaration type to haxe that acts like an interface, but allows default methods to be added. 

## Motivation

This is motivated by language systems Rust and Java. These languages support 
default implementations in traits and interfaces respectively. Given that default
implementations on interfaces have been rejected, this is the only remaining way 
that they can be implemented in the compiler. Currently, there are macros that allow
partial implementations, however they aren't, in my opinion, as good as a natural implementation, 
like Java or Rust.

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
name it will be a compiler error. 

JVM can also implement all Java interfaces as traits, which allows the compiler to drop the `@:jvm.default` metadata. 

When transpiling to targets, if the target already supports default implementations (like C#, Java, JVM), then no further work 
is needed. Otherwise, convert all traits into interfaces and copy the code into the classes implementing them. 
## Impact on existing code

As this is self contained, there shouldn't be breakage in code.

## Drawbacks

Having a new type declaration just to support default methods is a bit silly, 
however, as default methods on interfaces was rejected, this is the only way for it to work. 

## Alternatives

Simply adding default methods to interfaces would fix this, but this has already been rejected as a language feature.

There are also macros that acheive this, however other languages that have macros still have traits (see Rust, Scala)


## Opening possibilities

This could allow seamless Java/JVM interface support, which could allow the `@:jvm.default` metadata to be dropped

## Unresolved questions

I am unsure if I covered all of the inheritance possibilites (I likely didn't). Please point out any edge cases I forgot. 
