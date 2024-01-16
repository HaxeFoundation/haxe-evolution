# Package Alias

* Proposal: [HXP-0000](0000-package-alias.md)
* Author: [Matheus Dias de Souza](https://github.com/hydroper)

## Introduction

Allow an `import` pragma to lexically alias a package.

## Motivation

This feature is intended for increasing readability when working with package names.

In the present, an ambiguous reference can be resolved by using its package as a qualifier.

This feature allows the user to shorten a package qualifier and allows organizing imported items.

## Detailed design

Allow a `.*` sequence to appear in an aliasing `import` pragma before the `in` or `as` reserved word. Here is an example:

```haxe
import com.ea.n4s.* as n4s;

new n4s.Setup();
```

## Impact on existing code

This does not break compatibility.

## Drawbacks

There would be a drawback if the aliased name could be re-exported just like `typedef`, but it is just a "lexical" import. Therefore, it works on all targets.

## Alternatives

Regarding the current language, the only alternative to package aliases is to use the original package as qualifier. This proposal allows you to shorter a fully package qualified name.

The rationale for the `.*` token sequence in the introduced `import` is for distinguising packages and package properties.

## Opening possibilities

N/A

## Unresolved questions

N/A
