# Haxe change proposals

This project is for maintaining formal change proposals for Haxe language.

Stuff that don't need a formal proposal (unless it's something fundamental):

 * bugfixes
 * optimizations
 * documentation
 * minor API additions

Stuff that most probably need a formal proposal:

 * language changes, including adding new features
 * breaking standard library changes
 * large standard library additions (new toplevel types are also considered "large")
 * significant compiler architecture or build process changes

Basically, everything that needs some design process and consensus among the developers is a candidate for a proposal.

## The process (NOT YET FINALIZED)

 1. Fork the repo, copy the `0000-template.md` to `proposals/0000-short-name.md`,
    where `short-name` is a descriptive filename for the proposal document. Don't assign the number yet.
 2. Carefully fill in the proposal, pay attention to details: it should show your understanding
    of the issue and the impact of the proposed design.
 3. Submit a pull request with the proposal. In this PR, Haxe team and the community can provide
    feedback for it. Be prepared to react accordingly and revise the proposal document.
 4. After reaching general consensus, the PR is merged by someone from the Haxe developer team,
    and the proposal becomes "active". When merged, the proposal will receive its number from the
    corresponding pull request. If the proposal is rejected, the PR is closed with a comment explaining the reasons.
 5. Active proposals are to be further discussed in details by the Haxe developer team
    and finally implemented.
