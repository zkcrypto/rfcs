# zkcrypto RFCs - [RFC Book](https://zkcrypto.github.io/rfcs/)

[zkcrypto RFCs]: #zkcrypto-rfcs

The "RFC" (request for comments) process is intended to provide a consistent and
controlled path for changes to zkcrypto crates (such as new features) so that
all stakeholders can be confident about their direction.

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a
bit of a design process and produce a consensus among the zkcrypto community and
the crate maintainers.


## Table of Contents
[Table of Contents]: #table-of-contents

- [Opening](#zkcrypto-rfcs)
- [Table of Contents]
- [Crates in scope]
- [When you need to follow this process]
- [Before creating an RFC]
- [What the process is]
- [The RFC life-cycle]
- [Reviewing RFCs]
- [Implementing an RFC]
- [Reviewing and merging RFC implementations]
- [RFC Postponement]
- [Help this is all too informal!]
- [License]
- [Contributions]


## Crates in scope
[Crates in scope]: #crates-in-scope

The RFC process applies to the following Rust crates:

- [`ff`](https://github.com/zkcrypto/ff)
- [`group`](https://github.com/zkcrypto/group)
- [`pairing`](https://github.com/zkcrypto/pairing)
- [`bls12_381`](https://github.com/zkcrypto/bls12_381)
- [`jubjub`](https://github.com/zkcrypto/jubjub)
- [`pasta_curves`](https://github.com/zcash/pasta_curves)

To add or remove a crate from this list, a PR must be opened by a maintainer of
the crate (someone with the publish bit for the crate on [crates.io]). The
set of relevant maintainers of a crate in this list (and changes to the set)
must be made known to the maintainers of this RFC repository.


## When you need to follow this process
[When you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make "substantial" changes to
one of the crates listed above, or the RFC process itself. What constitutes a
"substantial" change is evolving based on community norms and varies depending
on what part of the ecosystem you are proposing to change, but may include the
following:

- Changes to the core zkcrypto traits.
- Changes or additions to the primary APIs of the crates (things that affect
  the API surfaces that downstream users are required to interact with).

Some changes do not require an RFC:

- Rephrasing, reorganizing, refactoring, or otherwise "changing shape does not
  change meaning".
- Additions that strictly improve objective, numerical quality criteria
  (speedup, better platform coverage, more parallelism, etc.)
- Additions only likely to be _noticed by_ other developers-of-zkcrypto-crates,
  invisible to users-of-zkcrypto-crates.

If you submit a pull request to implement a new feature without going through
the RFC process, it may be closed with a polite request to submit an RFC first.


## Before creating an RFC
[Before creating an RFC]: #before-creating-an-rfc

A hastily-proposed RFC can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected features, or those that don't fit
into the near-term roadmap, may be quickly rejected, which can be demotivating
for the unprepared contributor. Laying some groundwork ahead of the RFC can
make the process smoother.

Although there is no single way to prepare for submitting an RFC, it is
generally a good idea to pursue feedback from other zkcrypto ecosystem
developers beforehand, to ascertain that the RFC may be desirable; having a
consistent impact on the zkcrypto ecosystem requires concerted effort toward
consensus-building.

The most common preparations for writing and submitting an RFC include
discussing the topic on our [developer discussion forum], and occasionally
posting "pre-RFCs" on the developer forum. Issues filed on this repo should not
be used for pre-RFC content; they should generally either be about in-progress
RFCs with open PRs, tracking implementation of accepted RFCs, or bugs in
accepted RFCs.

As a rule of thumb, receiving encouraging feedback from long-standing zkcrypto
ecosystem developers, and particularly the relevant crate maintainers, is a good
indication that the RFC is worth pursuing.


## What the process is
[What the process is]: #what-the-process-is

In short, to get a major feature added to the zkcrypto crates, one must first
get the RFC merged into the RFC repository as a Markdown file. At that point the
RFC is "active" and may be implemented with the goal of eventual inclusion into
the relevant crates.

- Fork the RFC repo [RFC repository]
- Copy `0000-template.md` to `text/0000-my-feature.md` (where "my-feature" is
  descriptive). Don't assign an RFC number yet; This is going to be the PR
  number and we'll rename the file accordingly if the RFC is accepted.
- Fill in the RFC. Put care into the details: RFCs that do not present
  convincing motivation, demonstrate lack of understanding of the design's
  impact, or are disingenuous about the drawbacks or alternatives tend to
  be poorly received.
- Submit a pull request. As a pull request the RFC will receive design
  feedback from the larger community, and the author should be prepared to
  revise it in response.
- Now that your RFC has an open pull request, use the issue number of the PR
  to update your `0000-` prefix to that number.
- Each pull request will be labeled with the most relevant crates that it
  affects. An RFC repository maintainer will then assign the PR to a relevant
  crate maintainer.
- Build consensus and integrate feedback. RFCs that have broad support are
  much more likely to make progress than those that don't receive any
  comments. Feel free to reach out to the RFC assignee in particular to get
  help identifying stakeholders and obstacles.
- The maintainers of the relevant crates will discuss the RFC pull request, as
  much as possible in the comment thread of the pull request itself. Offline
  discussion will be summarized on the pull request comment thread.
- RFCs rarely go through this process unchanged, especially as alternatives
  and drawbacks are shown. You can make edits, big and small, to the RFC to
  clarify or change the design, but make changes as new commits to the pull
  request, and leave a comment on the pull request explaining your changes.
  Specifically, do not squash or rebase commits after they are visible on the
  pull request.
- At some point, a maintainer of one of the relevant crates will propose a
  "motion for final comment period" (FCP), along with a *disposition* for the
  RFC (merge, close, or postpone).
  - This step is taken when enough of the tradeoffs have been discussed that
    the relevant crate maintainers are in a position to make a decision. That
    does not require consensus amongst all participants in the RFC thread (which
    is usually impossible). However, the argument supporting the disposition on
    the RFC needs to have already been clearly articulated, and there should not
    be a strong consensus *against* that position outside of the relevant crate
    maintainers. Crate maintainers use their best judgment in taking this step,
    and the FCP itself ensures there is ample time and notification for
    stakeholders to push back if it is made prematurely.
  - For RFCs with lengthy discussion, the motion to FCP is usually preceded by
    a *summary comment* trying to lay out the current state of the discussion
    and major tradeoffs/points of disagreement.
  - Before actually entering FCP, at least one maintainer from each of the
    relevant crates must sign off; this is often the point at which many crate
    maintainers first review the RFC in full depth.
- The FCP lasts ten calendar days, so that it is open for at least 5 business
  days. It is also advertised widely, e.g. in
  [the Halo 2 Ecosystem Discord](https://discord.gg/wFB23QfH5r). This way all
  stakeholders have a chance to lodge any final objections before a decision
  is reached.
- In most cases, the FCP period is quiet, and the RFC is either merged or
  closed. However, sometimes substantial new arguments or ideas are raised,
  the FCP is canceled, and the RFC goes back into development mode.

## The RFC life-cycle
[The RFC life-cycle]: #the-rfc-life-cycle

Once an RFC becomes "active" then authors may implement it and submit the
feature as pull requests to the relevant crate repositories. Being "active" is
not a rubber stamp, and in particular still does not mean the feature will
ultimately be merged; it does mean that in principle all the major stakeholders
have agreed to the feature and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is "active" implies
nothing about what priority is assigned to its implementation, nor does it imply
anything about whether a zkcrypto ecosystem developer has been assigned the task
of implementing the feature. While it is not *necessary* that the author of the
RFC also write the implementation, it is by far the most effective way to see
an RFC through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted feature.

Modifications to "active" RFCs can be done in follow-up pull requests. We
strive to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect every
merged RFC to actually reflect what the end result will be at the time of the
next crate releases.

In general, once accepted, RFCs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes
should be new RFCs, with a note added to the original RFC. Exactly what counts
as a "very minor change" is up to the relevant crate maintainers to decide.


## Reviewing RFCs
[Reviewing RFCs]: #reviewing-rfcs

While the RFC pull request is up, the relevant crate maintainers may schedule
meetings with the author and/or relevant stakeholders to discuss the issues in
greater detail. A summary from the meeting will be posted back to the RFC pull
request.

The relevant crate maintainers make final decisions about RFCs after the
benefits and drawbacks are well understood. These decisions can be made at any
time, but the relevant crate maintainers should regularly issue decisions. When
a decision is made, the RFC pull request will either be merged or closed. In
either case, if the reasoning is not clear from the discussion in thread, the
relevant crate maintainers will add a comment describing the rationale for the
decision.


## Implementing an RFC
[Implementing an RFC]: #implementing-an-rfc

Some accepted RFCs represent vital features that need to be implemented right
away. Other accepted RFCs can represent features that can wait until some
arbitrary developer feels like doing the work. Every accepted RFC has an
associated issue in this repository tracking its implementation.

The author of an RFC is not obligated to implement it. Of course, the RFC
author (like any other developer) is welcome to post an implementation for
review after the RFC has been accepted.

If you are interested in working on the implementation for an "active" RFC, but
cannot determine if someone else is already working on it, feel free to ask
(e.g. by leaving a comment on the associated issue).

## Reviewing and merging RFC implementations
[Reviewing and merging RFC implementations]: #reviewing-and-merging-rfc-implementations

All implementations of RFCs need to go through a code review process that:

- upholds the software standards of the affected crates; and
- ensures alignment with the RFCs.

In general, a lead reviewer - preferably from a different community entity to
the lead implementer - will spearhead the review process. Depending on the kind
of RFC and/or the code it affects, additional reviewers may be required. For
instance, RFCs that affect consensus code (e.g. that would change ZKP circuits
or the validity of objects within some network) generally will require three
reviews on their implementations.

The final decision for merging an RFC implementation is made by the relevant
crate maintainers. They should ensure that the implementation has received
sufficient review to meet their software standards, and request additional
reviewers if necessary.


## RFC Postponement
[RFC Postponement]: #rfc-postponement

Some RFC pull requests are tagged with the "postponed" label when they are
closed (as part of the rejection process). An RFC closed with "postponed" is
marked as such because we want neither to think about evaluating the proposal
nor about implementing the described feature until some time in the future, and
we believe that we can afford to wait until then to do so. Postponed pull
requests may be re-opened when the time is right. We don't have any formal
process for that, you should ask people involved in the RFC and/or the crate
maintainers.

Usually an RFC pull request marked as "postponed" has already passed an informal
first round of evaluation, namely the round of "do we think we would ever
possibly consider making this change, as outlined in the RFC pull request, or
some semi-obvious variation of it." (When the answer to the latter question is
"no", then the appropriate response is to close the RFC, not postpone it.)


### Help this is all too informal!
[Help this is all too informal!]: #help-this-is-all-too-informal

The process is intended to be as lightweight as reasonable for the present
circumstances. We are trying to let the process be driven by consensus and
community norms, not impose more structure than necessary.


## License
[License]: #license

This repository is licensed under either of:

* Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or https://www.apache.org/licenses/LICENSE-2.0)
* MIT license ([LICENSE-MIT](LICENSE-MIT) or https://opensource.org/licenses/MIT)

at your option.


### Contributions
[Contributions]: #contributions

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.


[crates.io]: https://crates.io/
[developer discussion forum]: https://github.com/zkcrypto/rfcs/discussions
[RFC repository]: https://github.com/zkcrypto/rfcs
