# Haxe change proposals

This project is for maintaining formal change proposals for Haxe language.

## List of accepted proposals

| # | Name | Author | Status |
| --- | --- | --- | --- |
| `0001` | [Any type](proposals/0001-any.md) | [Dan Korostelev](https://github.com/nadako) | implemented in 3.3.0 |
| `0002` | [Arrow functions](proposals/0002-arrow-functions.md) | [Alexander Kuzmenko](https://github.com/RealyUniqueName) | implemented in 4.0.0 |
| `0003` | [New function type syntax](proposals/0003-new-function-type.md) | [Dan Korostelev](https://github.com/nadako) | implemented in 4.0.0 |
| `0004` | [Intersection types](proposals/0004-intersection-types.md) | [Simn](https://github.com/simn) | implemented in 4.0.0 |
| `0005` | [key => value iteration syntax](proposals/0005-key-value-iter.md) | [Dan Korostelev](https://github.com/nadako) | implemented in 4.0.0 |
| `0006` | [Inline markup](proposals/0006-inline-markup.md) | [Juraj Kirchheim](https://github.com/back2dos) | implemented in 4.0.0 |
| `0007` | [Module-level functions and variables](proposals/0007-module-level-funcs.md) | [Dan Korostelev](https://github.com/nadako) | [implemented in Haxe 4.2.0](https://github.com/HaxeFoundation/haxe/pull/8460) |
| `0008` | [Expression macro method forwarding](proposals/0008-macro-forward.md) | [Dan Korostelev](https://github.com/nadako) | [to be implemented](https://github.com/HaxeFoundation/haxe/issues/7453) |
| `0009` | [Inlining functions at call location](proposals/0009-inline-calls.md) | [YellowAfterlife](https://github.com/yellowafterlife) | implemented in 4.0.0 |
| `0010` | [New `asys` APIs](proposals/0010-asys.md) | [Aurel Bílý](https://github.com/Aurel300) | [to be implemented](https://github.com/Aurel300/haxe-sys) |
| `0011` | [Local bariable metadata syntax](proposals/0011-local-var-metadata.md) | [Peter Achberger](https://github.com/antriel) | [implemented in 4.2.0](https://github.com/HaxeFoundation/haxe/issues/9618) |
| `0012` | [Abstract classes](proposals/0012-abstract-classes.md) | [Aleksandr Kuzmenko](https://github.com/RealyUniqueName) | [implemented in 4.2.0](https://github.com/HaxeFoundation/haxe/pull/9716) |
| `0013` | [Default type parameters](proposals/0013-default-type-parameters.md) | [Ben Merckx](https://github.com/benmerckx) | [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10483) |
| `0014` | [Self access for abstracts](proposals/0014-self-access-for-abstracts.md) | [Mark Knol](https://github.com/markknol) | [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10482) |
| `0015` | [Local static variables](proposals/0015-local-static-variables.md) | [YellowAfterlife](https://github.com/yellowafterlife) | [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10477) |
| `0016` | [Null coalescing operator](proposals/0016-null-coalescing-operator.md) | [RblSb](https://github.com/RblSb) | [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10478) |
| `0017` | [Null-safe navigation operator](proposals/0017-null-safe-navigation-operator.md) | [Robert Borghese](https://github.com/RobertBorghese) | [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10479) |
| `0018` | [Number separators](proposals/0018-number-separators.md) | [Marcelo Silva Nascimento Mancini](https://github.com/MrcSnm) | [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10480) |
| `0019` | [Numeric literal suffixes](proposals/0019-numeric-iteral-suffixes.md) | [Aidan Lee](https://github.com/aidan63) | [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10481) |

## About proposals

Haxe Proposals (HXP) can be submitted by any Haxe team developer or community member, as long as it's complete and well explained (please see the "How to submit an HXP" section below).

Once a HXP has been submitted, it can be discussed and modified. Anybody can also submit a PR against an existing HXP to have it amended.

> Please understand that the HXP discussion and voting process is for
> cases where the core team is not opposed to the proposed
> changes. The team internally discusses all proposals; if they are
> opposed to a given HXP then it may be rejected with very little
> public discussion in the PR comments. There is simply not enough
> time for the core team to provide detailed written rationale for
> every proposed change which they think would not be a good overall
> fit for Haxe.

After the HXP has been discussed, a formal vote can take place to accept it. Only Haxe Core Team members are allowed to vote:

 - Nicolas Cannasse [@ncannasse](https://github.com/ncannasse) (Haxe creator)
 - Simon Krajewski [@Simn](https://github.com/Simn) (Compiler Maintainer)
 - Hugh Sanderson [@hughsando](https://github.com/hughsando) (Haxe C++ backend)
 - Aurel Bílý [@aurel300](https://github.com/Aurel300) (Haxe contributor)
 - Andy Li [@andyli](https://github.com/andyli) (tools and infrastructure)
 - Dan Korostelev [@nadako](https://github.com/nadako) (compiler contributor)
 - Alexander Kuzmenko [@RealyUniqueName](https://github.com/RealyUniqueName) (Compiler Maintainer)
 - Justin Donaldson [@jdonaldson](https://github.com/jdonaldson) (Haxe Lua backend)

To be accepted the HXP needs half + 1 votes in favor of it.

In order to keep the long term goals and vision of Haxe, Nicolas can veto any accepted proposal after the vote, but will explain his reasoning for doing so in details and agrees to use this power with care.

If the HXP is accepted, the core team will work on implementing it.

## What needs a proposal?

Stuff that doesn't need a formal proposal (unless it's something fundamental):

 * bugfixes
 * optimizations
 * documentation
 * minor API additions

Stuff that most probably needs a formal proposal:

 * language changes, including adding new features
 * breaking standard library changes
 * large standard library additions (new toplevel types are also considered "large")
 * significant compiler architecture or build process changes

Basically, everything that needs some design process and consensus among the developers is a candidate for a proposal.

## How to submit an HXP

 1. Fork the repo, copy the `0000-template.md` to `proposals/0000-short-name.md`,
    where `short-name` is a descriptive filename for the proposal document. Don't assign the number yet.
 2. Carefully fill in all sections of the proposal. Pay attention to details: it should show your understanding
    of the issue and the impact of the proposed design.
 3. Submit a pull request with the proposal. In this PR, Haxe team and the community can provide
    feedback for it. Be prepared to react accordingly and revise the proposal document.
 4. After reaching general consensus or voting takes place, the PR is merged by someone from the Haxe developer team,
    and the proposal becomes "active". When merged, the proposal will receive its number from the
    corresponding pull request. If the proposal is rejected, the PR is closed with a comment explaining the reasons.
 5. Active proposals are to be further discussed in details by the Haxe developer team
    and finally implemented.

## Voting process

It's the responsibility of the author of a proposal to start the voting process.

Once discussion is exhausted, the author of a proposal can request the voting.
Following requirements should be met:

* All questions asked by the core members of Haxe Foundation to the author are answered;
* The last edit to a proposal was made at least two weeks ago;
* The last message in the discussion was posted at least two weeks ago.

To start the voting process the author should post the comment:

> Request for voting.

Additionally, the author can tag voters in this comment to draw their attention ([example](https://github.com/HaxeFoundation/haxe-evolution/pull/48#issuecomment-412341110)).

After this comment is posted the discussion will be locked by any Haxe Foundation member with write access to the haxe-evolution repository. From that point only voters will be able to post comments to the discussion.
