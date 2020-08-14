# Single pattern checks

* Proposal: [HXP-NNNN](NNNN-single-pattern-checks.md)
* Author: [Dmitrii Maganov](https://github.com/vonagam)

## Introduction

Proposal to rename `.match` single pattern check to `.case`.

## Motivation

To support usage on any type, not only on enum values.

## Detailed design

```haxe
value.case(pattern) // returns a boolean
```

`.match` can be kept as an alias which is limited to enum values or depreciated.
