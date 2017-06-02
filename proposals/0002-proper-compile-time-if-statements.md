# Any type

* Proposal: [HXP-0002](0002-proper-compile-time-if-statements.md)
* Author: [Laurence Taylor](https://github.com/0b1kn00b)

## Introduction

Provide a way to use #if statements where library names contain non-identifier literals.

## Motivation

At the moment, haxelibs *cannot* contain the `-` character, which means that you cannot use them with `#if... #end` statements, meaning that if you want to have interoperability code with these libraries, you have to include them as dependencies, whether or not they are needed.

## Detailed design

allow:

    #if "thx.core"
    #end

## Drawbacks

  Can't see any, the `#if` statements seem to be an edge case as they operate at compile time, so no massive rewrites, I think.

## Alternatives

    #if some-lib
    #end

## Opening possibilities
