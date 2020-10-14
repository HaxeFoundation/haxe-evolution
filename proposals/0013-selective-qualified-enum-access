# ::Selective Qualified Enum Access::
* Proposal: [HXP-0013](0013-selective-qualified-enum-access.md)
* Author: [Haxe Developer](https://github.com/0b1kn00b)

## Introduction

Allow control over the publishing of Enum constructors in the global scope.

## Motivation

Namespacing is important because it allows natural and comprehensible names within any given scope. 

Enum constructors break with this convention for the sake of brevity, but can cause problems when integrating several grammars and publishing artifacts. When publishing a programming library that requires the use of enums it's necessary to work around this flaw. 

Given, that for simplicity sake, it's polite to require as few imports as possible; naming conventions must be applied so as not to cause aliasing, which nullifies the advantages of namespaces.

Without those naming conventions, it's necessary to push enumerations into separate packages and then explain somewhere where the missing piece is.

#### tldr:

It should be possible to suggest that enum constructors be used qualified so as to respect the global scope.

## Detailed design

Using a metadata tag `@:qualified_enum_access` should cause unqualified references to it's constructors to be inaccessable. `Enum.Constructor` remains valid. 

Possible override in the same manner of `@:access`: `@:unqualified_enum_access`

## Impact on existing code
  None I can think of.

## Drawbacks
  It would make some pattern matching code more verbose if the override isn't implemented, although `import LongEnumName in L` isn't too bad.

## Alternatives
  It's possible to get around it with boilerplate: putting the enum in a different package and having a class of static constructors available in the entry point.

## Opening possibilities

## Unresolved questions
  Is it a pita to implement?
