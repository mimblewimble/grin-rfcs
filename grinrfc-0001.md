# GRIN-RFC-0001 Grin RFC Process

```
- Number: GRIN-RFC-0001
- Title: Grin RFC Process
- Status: Draft
- Authors: joltz (joltz@protonmail.com)
- Created : June 21st, 2019
```

# Summary
[summary]: #summary

The "RFC" (request for comments) process is intended to provide a consistent and controlled path for new features to enter the Grin codebase and governance structure, so that all stakeholders can be confident about the direction the project is evolving in.

# Motivation
[motivation]: #motivation

The previous way of adding new features to Grin has been good for early development, but for Grin to become a mature project and ecosystem it needs structure and transparency when it comes to changing the system. This is a proposal for a more principled RFC process to make it a more integral part of the overall development and governance processes, and one that is followed consistently to introduce features to Grin.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

To add a major feature to Grin, one must first get the RFC merged into the RFC repo as a markdown file. At that point the RFC is 'active' and may be implemented with the goal of eventual inclusion into Grin.

## When you need to follow this process

You need to follow this process if you intend to make "substantial"
changes to the Grin codebase or governance process. What constitutes a "substantial"
change is evolving based on community norms, but may include the following.

   - Any semantic or syntactic change to the wallet, node, miner or crypto library that is not a bugfix.
   - Major changes in ecosystem content such as the docs, site or explorer
   - Removing Grin features, including those that are feature-gated.

Some changes do not require an RFC:

   - Rephrasing, reorganizing, refactoring, or otherwise "changing shape
does not change meaning".
   - Additions that strictly improve objective, numerical quality
criteria (warning removal, speedup, better platform coverage, more
parallelism, trap more errors, etc.)
   - Additions only likely to be _noticed by_ other developers-of-grin,
invisible to users-of-grin.

If you submit a pull request to implement a new feature without going
through the RFC process, it may be closed with a polite request to
submit an RFC first.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

* Fork the RFC repo https://github.com/mimblewimble/grin-rfcs
* Copy `grinrfc-0000.md` to `grinrfc-0000-my-feature.md` (where
'my-feature' is descriptive. don't assign an RFC number yet).
* Fill in the RFC
* Submit a pull request. The pull request is the time to get review of
the design from the larger community.
* Build consensus and integrate feedback. RFCs that have broad support
are much more likely to make progress than those that don't receive any
comments.

Eventually, somebody on the assigned subteam will either accept the RFC by
merging the pull request, at which point the RFC is 'active', or
reject it by closing the pull request.

Whomever merges the RFC should do the following:

* Assign an id, using the PR number of the RFC pull request. (If the RFC
  has multiple pull requests associated with it, choose one PR number,
  preferably the minimal one.)
* Add the file in the `text/` directory.
* Create a corresponding issue on [Grin-PM repo](https://github.com/mimblewimble/grin-pm)
* Fill in the remaining metadata in the RFC header, including links for
  the original pull request(s) and the newly created Grin-PM issue.
* Add an entry in the Active RFC List of the root `README.md`.
* Commit everything.

Once an RFC becomes active then authors may implement it and submit the
feature as a pull request to the relevant Grin repo. An 'active' is not a rubber
stamp, and in particular still does not mean the feature will ultimately
be merged; it does mean that in principle all the major stakeholders
have agreed to the feature and are amenable to merging it.

Modifications to active RFC's can be done in followup PR's. An RFC that
makes it through the entire process to implementation is considered
'complete' and is removed from the [Active RFC List]; an RFC that fails
after becoming active is 'inactive' and moves to the 'inactive' folder.

# Drawbacks
[drawbacks]: #drawbacks

* May not be formal enough
* May not encourage sufficient community engagement
* May slow down needed features
* May allow some features to be included too quickly

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Alternatively retain the current informal RFC process. The newly proposed RFC process is
designed to improve over the informal process in the following ways:

* Discourage unactionable or vague RFCs
* Ensure that all serious RFCs are considered equally
* Improve transparency for how new features are added to Grin
* Give confidence to those with a stake in Grin's development that they
understand why new features are being merged

As an alternative, we could adopt an even stricter RFC process than the one proposed here. If desired, we should likely look to Bitcoin's BIP or Python's PEP process for inspiration.

# Prior art
[prior-art]: #prior-art

Most decentralized cryptocurrency projects have adopted an RFC-like process to manage adding new features.

Bitcoin uses BIPs which are an adaptation of Python's PEPs. These processes are similar to the Rust RFC process which has had success in the Rust community as well as in other cryptocurrency projects like Peercoin.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. Does this RFC strike a favorable balance between formality and agility?
2. Does this RFC successfully address the aforementioned issues with the current
   informal process?
3. Should we retain rejected RFCs in the archive?
4. Should RFC issues be opened in their respective repos (wallet RFC in wallet repo etc.) or should they all be opened in grin-pm repo?

# Future possibilities
[future-possibilities]: #future-possibilities

This proposal was initially based on an RFC process for codebase development.
As the process evolves it will have a larger impact in the governance of Grin.
This is a relatively new area of exploration as governance processes can have
wide ranging impacts on the ecosystem as a whole.

Just as it is important to hone the language to support the development process
and life-cycle it is also important to sharpen the language to support governance
processes and life-cycles for the Grin ecosystem.
