# Number Separators

* Proposal: [0013-number-separator](0013-number-separator.md)
* Author: [Hipreme|Marcelo Silva Nascimento Mancini](https://github.com/MrcSnm)

## Introduction

Make the following syntax available: 1_000_000 as an alternative for 1000000

## Motivation

I believe that number separator is achievable with macros, although I'm
pretty sure that it would unnecessarily increase compile time and it is
something pretty simple to implement.

D and Javascript are successful languages which contain those features,
the make a lot easier to read when working with great numbers, as transforming
from nanoseconds to seconds: 1 / 1000000000. It is fairly impossible to read
that number and don't count the zeros to know which number is that, with separators,
we can make triples: 1_000_000_000, then you will check that exists 3 triples of 0.

The same thing applies to binary, which can be pretty overwhelming, specially if one
decides to use 32 bit binaries:

0b00000000000000000000000000000000
vs
0b0000_0000_0000_0000_0000_0000_0000_0000_0000

Although unusual, it is possible to happen, and it will make the code much harder to
read.

## Detailed design

When detecting numbers, just check if there is a separator, if it is, just continue
the process of number parsing. I believe the only corner case is from uses trying
to do bad things: 1000_ : Using the separator as the last string to parse could
just send an error.

## Impact on existing code

None.

## Drawbacks

None.

## Alternatives

Macros would make a redundant string parsing, plus having a function to call for
making that syntax available would break the visibility, which is the main reason
for that proposal.


## Unresolved questions

None.
