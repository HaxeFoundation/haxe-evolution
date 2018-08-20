# Haxe Evolution voting process

* Proposal: [HXP-NNNN](NNNN-voting-process.md)
* Author: [Alexander Kuzmenko](https://github.com/RealyUniqueName/)

## Introduction

Adjustment to Haxe Evolution voting process.

## Motivation

Currently a lot of proposals are stalled because of discussions seem exhausted, but no one knows when (and if) it's time for the voting process to start.
This proposal aims to eliminate such an undefined state of evolution proposals.

## Detailed design

Author of a proposal (Author) gets the right to start the voting process.
And now it's Author's responsibility to start it if Author is satisfied with the state of a proposal and the following conditions are met:

* All questions asked by Voters to Author are answered;
* The last edit to a proposal was made at least two weeks ago;
* The last message in the discussion was posted at least two weeks ago.

To request a voting Author should post a comment with the following text:

> Request for voting.

Additionally Author can tag Voters in that comment to draw their attention.

After seeing such a comment any Haxe Foundation member with the write access to the haxe-evolution repository should lock the conversation (special button in the right panel in PR UI on GiHub)

To be able to post their votes after conversation is locked Voters should be given the write access to the haxe-evolution repository.