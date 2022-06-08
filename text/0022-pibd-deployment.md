
- Title: pibd-deployment
- Authors: [Michael Cordner (Yeastplume)](mailto:mc@revcore.net)
- Start date: June 6, 2022
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [mimblewimble/grin#3695]

---

## Summary
[summary]: #summary

This RFC describes the rollout of Parallel Independent Block Download (PIBD) functionality on the Grin network. The aim of this RFC is to:

1) Formalize the timing of phased releases that enable PIBD on the network and retire the earlier `txhashset.zip` method of fast-syncing new nodes.
2) Provide implementation details and background technical descriptions that are not provided in the original [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md)

## Motivation
[motivation]: #motivation

Previously, the fast-sync process for bringing new nodes onto the network involved downloading and requesting the entire UTXO set up to a 'horizon' block, below which re-orgs are deemed to be no longer be a concern. This is transmitted by a single randomly-chosen peer in a single `.zip` file (known as the `txhashset.zip` file). Problems with this approach include (but are not limited to):

* The UTXO set is retrieved from a single peer as opposed to many peers on the network
* The download speed of the txhashset is limited to the speed of the randomly chosen peer, leading to wildly different node sync times in practice.
* Downloads must be restarted if interrupted
* Nodes responding to PIBD requests exhibit a noticable pause as a result of needing to mutex lock and zip the entire UTXO set when requested.

PIBD is a new fast-sync method that leverages the unique structure of Grin's backend PMMRs to break off and transmit 'segments' of Grin's UTXO set independently of each other. This method ensures that

* The UTXO set is received from multiple peers in chunks, then validated and pieced together by the syncing node
* The process can be stopped and resumed at will
* The time to process and sync a new node should be more consistent. (Note this doesn't necessarly mean 'faster' in all cases)

Details of the PIBD message protocol, segments and proof generation are defined within the [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md). This RFC will not repeat information found there, but will focus on implementation details that aren't covered therein.

## Reference-level explanation

### Segmentation, segment creation and responding to PIBD segment request messages

#### Capabilities Flag

Nodes capable of responding to PIBD segment request messages advertise the flag `PIBD_HIST_1` denoted by hex `0x40` (binary `0b0100_0000`) in their capabilities field.

Note that there is a previous flag in the core implementation `PIBD_HIST` denoted by hex `0x10` (binary `0b0001_0000`). While this flag was rolled out along with segmentation capabilities in grin-core 5.0 in anticipation of a full PIBD implementation, a bug in the original segmentation code unfortunately means this flag is unusable. See https://github.com/mimblewimble/grin/pull/3705 for further details.

#### Segment creation

A node responds to a PIBD segment request by creating a segment of an underlying PMMR along with a merkle proof demonstrating inclusion in the complete PMMR up to a position represented by the current horizon header. Note that the responding node must always respond with segments corresponding to the consensus-agreed horizon header (and it is not possible for a nodes to explicitly request a segment for any other header).

#### Horizon Header Height

The horizon header height is a consensus-derived agreed header below which the UTXO set is deemed to be outside of the possiblity of re-org. (This is the header on which the original `txhashset.zip` method is based). This can simply be thought of as 'the UTXO set as it stood as of a certain block'.

The current horizon header height is determined by subtracting a state sync threshold (2 'days' worth of grin blocks) from the current header height. This height is then 'rounded down' to a height representing the start of a 12 'grin-block-hour' window, which remains the horizon header for that 12 hour window. The exact calculation of this height is:

```
DAY_HEIGHT: 1440
STATE_SYNC_THRESHOLD = @ 2 * DAY_HEIGHT = 2880
ARCHIVE_INTERVAL = 12*60 = 720

THRESHOLD_HEIGHT = CURRENT_HEADER_HEIGHT - STATE_SYNC_THRESHOLD
HORIZON_HEADER_HEIGHT = THRESHOLD_HEIGHT - (THRESHOLD_HEIGHT % ARCHIVE_INTERVAL)
```

e.g.

```
CURRENT_HEIGHT = 1_000_000
THRESHOLD_HEIGHT = 1_000_000 - 2880 = 997120
HORIZON_HEADER_HEIGHT = 997120 - (977120 % 720) = 99640
```

#### PMMR Size Selection and Proof Generation

Nodes responding to PIBD message requests must only respond with with segments and proofs of inclusion for the PMMR of the size represented within the current horizon header. These sizes are defined within the header as:

```
output_mmr_size: Output and Rangeproof segment requests

kernel_mmr_size: Kernel segment requests
```

Merkle proofs within a segment must prove inclusion in a PMMR as follows:

```
Bitmap segment requests: The root of the bitmap PMMR at the given height hashed against the root of the output pmmr. When combined, this should give the value of `output_root` in the header.

Output segment requests: The root of the output PMMR at the given height hashed with the root of the output pmmr. When combined, this should give the value of `output_root` in the header.

Rangeproof Segment Requests: The root should correspond directly to the header field `range_proof_root`

Kernel Segment Requests: The root should correspond directly to the header field `kernel_root`
```

Note that it is not possible for a nodes to explicitly request a segment for a particular header; it's assumed that the request is always for the current horizon header.

#### PMMR Rewinding

Segments must be created against the UTXO set as it existed at the horizon header, which means that all prunable PMMRs (i.e. output and rangeproof) PMMRs must be rewound to ensure the leaf inclusion set is as it existed at that block. In practice, implementations only need to rewind the entire txhashset once (on initialisation of the segmenting code or when the horizon block changes), and maintain a representative bitmap set to use in for subsequent PIBD segment requests.

### Requesting Segments and Recreating the UTXO Set

Intro... this must happen on the client side
   * determine horizon block
   * request current bitmap set
   * Determine which segments to request
   * determine what you have

* Desegmentation

* Segment sizes

* When to give up


* Order of validation/verification, explanation 

* we check pruning of entire set is correct before validation, leaf set may have changed

* What happens when horizon window rolls over (should be able to resume from given data, with new bitmap set, but bitmap set may have changed, we now need to check for and prune outputs that may have been spent since) (which we do before validation)

* Resumption after a period of inactivity.


* Existing impls should have a flag in their config files via which it can be turned off

* Segments can contain spent objects, must be pruned as tree is being built, optimisation for future updates.

* May store the last verified rangeproof/kernel/etc for verification


### Rollout Timing

* Alpha released containing full PIBD implementation for testing (testnet only) - Apr 20th, 2022
* Target date for reviewing and merging work performed in `pibd_impl` into master, enabling PIBD on current master for testnet only - July 5th, 2022
* Release of 5.2.0, containing merged PIBD work on testnet only (and a few other features TBD) - Sept 1st 2022
* Release of 5.2.1 - turning on PIBD sync for mainnet as well as testnet - Feb 1st 2023
* Version as yet undetermined: remove support for earlier `txhashset.zip` method of syncing - Feb 1st 2024













## Community-level explanation
[community-level-explanation]: #community-level-explanation

Explain the proposal as if it were already introduced into the Grin ecosystem and you were teaching it to another community member. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Grin community members should *think* about the improvement, and how it should impact the way they interact with Grin. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Grin community members and new Grin community members.

For implementation-oriented RFCs (e.g. for wallet), this section should focus on how wallet contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

## Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For core, node, wallet and infrastructure proposals: Does this feature exist in other projects and what experience have their community had?
- For community, ecosystem and moderation proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other projects.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that Grin sometimes intentionally diverges from common project features.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the project and ecosystem as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how it fits into the road-map of the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.

## References
[references]: #references

Include any references such as links to other documents or reference implementations.

- [reference 1](#references)
- [reference 2](#references)
