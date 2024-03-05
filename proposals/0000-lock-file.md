# Improvements to Per-Project Haxelib Setups

* Proposal: [HXP-0000](0000-lock-file.md)
* Author: [EliteMasterEric](https://github.com/EliteMasterEric)

## Introduction

This proposal suggests reworks and improvements to `haxelib` to improve the ability to install specific library versions required for a project, even when the user is implementing frameworks which would otherwise interfere with the ability of `haxelib` to query the status of a project's dependencies.

## Motivation

Maintaining a versionable and replicable state for any project's dependencies is extremely important, as it means the difference between being able to explore and use older code and said code being completely unusable.

Notably, `haxelib install all` exists to solve this problem. This provides a means through which Haxelib can scan the HXMLs in a project folder, extract the hardcoded library versions, and install them for you automatically. This is done at a global level, which means it is done automatically.

However, this approach touted on the [HaxeLib documentation](https://lib.haxe.org/documentation/per-project-setup/) has a major flaw; it is predicated on the use of `HXML` by a project. Many frameworks such as Lime do not include HXMLs in their project folders at all, instead utilizing the Haxe compiler directly (or generating one to build with, but that mandates that the correct framework version is ), either do not utilize HXMLs in their projects at all, or only utilize it as a temporary file during the build step, and in either case no `HXML` is included in the project folder for use.

Thus, it is necessary that a separate file, dedicated to defining the list of project dependencies and providing the ability to pin a given project to exact dependency versions. This is similar behavior to things such as [NodeJS's package-lock](https://docs.npmjs.com/cli/v10/configuring-npm/package-lock-json) and [Ruby's Gemfile.lock](https://medium.com/never-hop-on-the-bandwagon/gemfile-and-gemfile-lock-in-ruby-65adc918b856).

## Detailed design

TBD

## Impact on existing code

As this would involve reworks to Haxelib, there should be no impact on existing codebases. The reworked implementation of `haxelib install all` should fall back to the existing behavior for the purposes of maintaining scripts (possibly with a deprecation warning).

## Drawbacks

Aside from the labor of development, I do not anticipate any issues with this proposal.

## Alternatives

The primary existing alternative to provide this functionality in Haxe is [hmm](https://github.com/andywhite37/hmm/), a CLI tool which calls Haxelib to perform its operations. It creates and maintains a `hmm.json` file, which stores information on which libraries the project has installed and allows reinstallation of those libraries, and operates via a local repo.

This implementation has several problems, among them the fact it mandates the use of a local repo, which prevents reuse of library files, as well as the fact that `hmm` is itself a Haxelib, causing several bugs when operating it. Additionally, since it is not a core part of Haxe, users must be instructed to download it before use.

## Unresolved questions

Currently, the actual design and operation of a new lockfile format is still to be determined.
