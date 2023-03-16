# Package Alias

* Proposal: [HXP-0000](0000-package-alias.md)
* Author: [Matheus Dias de Souza](https://github.com/hydroper)

## Introduction

Allow an `import` pragma to lexically alias a package.

## Motivation

This feature is intended for increasing readability when working with package qualified names.
Currently an ambiguous reference can be resolved by fully-qualifying its package.
This feature allows to shorter or lexically rename what you use to qualify a reference.
Even if a lexical reference is not ambiguous, aliasing a package and using that alias as a qualifier
instead of a lexical reference can be useful for giving an idea of what a reference is.

## Detailed design

Allow a `.*` sequence to appear in an aliasing `import` pragma before the `in` or `as` reserved word. Here is an example:

```haxe
import ecma262.temporal.* as Temporal;

var zdt = new Temporal.ZonedDateTime({});
```

## Impact on existing code

This does not break compatibility.

## Drawbacks

There would be a drawback if the aliased name could be re-exported just like `typedef`, but it is just a "lexical" import;
therefore it works on all targets.

## Alternatives

The only alternative is to use the original package as qualifier. This proposal allows you to shorter a fully-qualified package.

## Opening possibilities

N/A

## Unresolved questions

N/A
