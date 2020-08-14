# Destructuring assignments

* Proposal: [HXP-NNNN](NNNN-destructuring.md)
* Author: [Dmitrii Maganov](https://github.com/vonagam)

## Introduction

Syntax for safe/non-throwing and unsafe/throwing destructuring assignments.

## Motivation

Destructuring assignments is quite common feature in programming languages that help with readability.

## Detailed design

Not to mess with existing variables declaration expression (so not to complicate a parser and not to deal with commas) and because of syntax that haxe patterns can use (variable declarations and different assignments) decided to go with new not taken yet operator:

```haxe
Some(var int) =< option; // will throw if option is None

if (Some(var int) =< option) trace(int);
```

To addess concerns about "variables appear without declaration" it will be made mandatory to mark variables with `var`/`final` if you want to access them outside of a pattern. (Need to allow `var name = pattern` as pattern for that, easy to do, strange that it have not be done yet.)

Operator looks like a funnel placed to pour a value into a pattern, `EFunnel` for expression name then. Should have a precedence lower than arrow one since last one is used in extractors.
