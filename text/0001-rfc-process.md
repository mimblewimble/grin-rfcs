# 0001-rfc-process

# THIS RFC IS STILL IN DRAFT MODE

- Title: rfc-process
- Authors: [joltz](mailto:joltz@protonmail.com), [yeastplume](mailto:yeastplume@protonmail.com), [lehnberg](mailto:daniel.lehnberg@protonmail.com)
- Start date: June 21st, 2019

---

# Summary
[summary]: #summary

The "RFC" (request for comments) process is intended to provide a consistent and controlled path for improvements to be made to Grin.

# Motivation
[motivation]: #motivation

Many changes, including bug fixes and documentation improvements can be implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and could benefit from being put through a bit of a design process in order to produce a consensus among Grin community participants and stakeholders.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

To add a major feature to Grin or make a change to Grin's governance structure, one must first get the RFC merged into the RFC repo as a markdown file. 
At that point the RFC is 'active' and may be implemented with the goal of eventual inclusion into the Grin codebase or Governance procedures.

## When you need to follow this process

You need to follow this process if you intend to make "substantial"
changes to the Grin codebase or governance process. What constitutes a "substantial"
change may evolve based on community norms, but may include the following.

   - Any semantic or syntactic change to the wallet, node, miner or underlying crypto libraries that is not a bugfix.
   - Major changes in ecosystem content such as the docs, site or explorer
   - Removing Grin features, including those that are feature-gated.

Some changes do not require an RFC:

   - Rephrasing, reorganizing, refactoring, or changes that are not
 visible to Grin's users.
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

1. Fork the RFC repo https://github.com/mimblewimble/grin-rfcs
2. Copy `0000-template.md` to `text/0000-my-feature.md` (where
'my-feature' is descriptive. don't assign an RFC number yet).
1. Write the RFC according to the template instructions.
1. Submit a pull request.
1. The relevant team (sub-team or core) will perform an initial review of the PR, and decide whether the RFC should be merged into the repository as a Draft PR.

Once the PR is accepted into the repository, it will be given 'Draft' status. This does not mean the RFC has been accepted, only that is has been
selected by the community for further discussion, feedback and refinement.

At this point, whoever merges the PR should:
* Assign an id, using the PR number of the RFC pull request. (If the RFC has multiple pull requests associated with it, choose one PR number,
  preferably the minimal one.)
* Rename the file to `grin-rfc-xxxx.md` and merge into `/text`. (where
xxxx is the RFC ID)
* Fill in the remaining metadata in the RFC header.
* Add an entry in the RFC List in `README.md`, with a status of Draft.

When the RFC is in 'Draft' stage, it is important to refine details, build consensus among the community for the RFC and integrate useful
feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments.

Eventually, the relevant community organization will either accept the RFC by changing its status to 'Accepted' or reject it by setting it to 'Rejected'.

Once an RFC becomes active then authors may implement it and submit the feature as a pull request to the relevant Grin repo. An 'Accepted' status is not a rubber stamp, and in particular still does not mean the feature will ultimately
be merged; it does mean that in principle all the major stakeholders have agreed to the feature and are amenable to merging it.

Modifications to Accepted RFC's can be done in followup PR's. An RFC that makes it through the entire process to implementation is considered
'Active'; an RFC that fails after becoming active is 'Inactive' and is relabelled as such.

# Drawbacks
[drawbacks]: #drawbacks

* May not be formal enough
* May not encourage sufficient community engagement
* May slow down needed features
* May allow some features to be included too quickly

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Alternatively retain the current informal RFC process. The proposed RFC process is designed to improve over the informal process in the following ways:

* Discourage unactionable or vague RFCs
* Ensure that all serious RFCs are considered equally
* Improve transparency for how new features are added to Grin
* Give confidence to those with a stake in Grin's development that they
understand why new features are being merged
* Assist the Grin community with feature and release planning.

As an alternative, we could adopt an even stricter RFC process than the one proposed here. If desired, we should likely look to Bitcoin's BIP or Python's PEP process for inspiration.

# Prior art
[prior-art]: #prior-art

Most decentralized cryptocurrency projects have adopted an RFC-like process to manage adding new features.

Bitcoin uses BIPs which are an adaptation of Python's PEPs. These processes are similar to the Rust RFC process which has had success in the Rust community as well as in other cryptocurrency projects like Peercoin.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. Does this RFC strike a favorable balance between formality and agility?
2. Does this RFC successfully address the aforementioned issues with the current informal process?
3. Should we retain rejected RFCs in the archive?  (YP: I think yes, so it's apparent to new submitters if their idea has already been considered and rejected)
4. Should RFC issues be opened in their respective repos (wallet RFC in wallet repo etc.) or should they all be opened in grin-pm repo? (YP: I think 1 repository dedicated to Grin RFCs is fine)

# Future possibilities
[future-possibilities]: #future-possibilities

This proposal was initially based on an RFC process for codebase development.
As the process evolves it will have a larger impact in the governance of Grin.
This is a relatively new area of exploration as governance processes can have
wide ranging impacts on the ecosystem as a whole.

Just as it is important to hone the language to support the development process
and life-cycle it is also important to sharpen the language to support governance
processes and life-cycles for the Grin ecosystem.

# References
[references]: #references

https://github.com/rust-lang/rfcs

https://github.com/bitcoin/bips/blob/master/bip-0001.mediawiki

https://www.python.org/dev/peps/pep-0001/
