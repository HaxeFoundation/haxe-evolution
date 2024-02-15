# ::enter feature name here::

* Proposal: [HXP-0020](0020-macro-sensitive-imports.md)
* Author: [0b1kn00b](https://github.com/ohmrun)

## Introduction

Cleaen imports without compiler switches

## Motivation

An error that often baffles me is where a type has been implemented at macro time but is also imported 
at macro time and therefore compilation fails. Careful switches have to be placed and it's usability
minefield.

## Detailed design

Require imports to be qualified as `@:macro`, either in definitions (i.e `std`) or declaration side:

```haxe 
@:macro import some.thing.i.NeedForMacros
```

This makes macro code more forgiving, and less error prone to badly placed switches and hunting through import dependency chains as it makes explicit what is needed in the macro context.

## Impact on existing code

I can't think of a backwards compatible version of this at the moment, but ports shouldn't be too much work given schlepping through dependency chain scope issues is something that macro writers already have to know how to do.

## Drawbacks

Tell me.

## Alternatives

I can do `#if macro #end` switches for every program that needs macros.

There might be a folder based version or a file extension version, I'm not sure.

## Opening possibilities

## Unresolved questions

What to you guys think?
