
- Title: variable-size-kernels
- Authors: [Antioch Peverell](mailto:apeverell@protonmail.com)
- Start date : Aug 13, 2019
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

We propose minimizing the serialized kernel data by only including data applicable for each kernel feature variant.
i.e. Only include a lock_height on height locked kernels and do not include a fee on coinbase kernels.

# Motivation
[motivation]: #motivation

We currently include both the fee and the lock_height on _every_ kernel in binary serialization format.
We store a fee of 0 on coinbase kernels and a lock_height of 0 on plain and coinbase kernels.
Kernels are never pruned and exist forever. This data is unnecessary and must be stored indefinitely and
sent to peers during initial fast sync.
This will reduce the total size of the kernels in binary serialized format, reducing local storage costs and reducing
the amount of data necessary for initial fast sync.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

Transaction kernels currently have several pieces of "optional" data that are only applicable for particular kernel feature variants. Coinbase kernels do not have an associated fee and height locked kernels are the only kernels that have an associated lock_height. We currently implement these by always including these values but using 0 if not applicable.

These are both serialized as 8 bytes (u64) making serialization/deserialization simple but at the cost of unneccessary data that is effectively ignored.

As the number of kernels continues to grow over time this unneccessary data becomes ever more expensive.

By not serializing a zero value for the lock_height on _every_ kernel we will save 8 bytes per kernel. This may not seem significant but this saves 8MB for one million plain kernels.

The trade-off is more complex serialization/deserialization rules and additional complexity in terms of backward compatibility across the network to roll this change out.




Explain the proposal as if it were already introduced into the Grin ecosystem and you were teaching it to another community member. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Grin community members should *think* about the improvement, and how it should impact the way they interact with Grin. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Grin community members and new Grin community members.

For implementation-oriented RFCs (e.g. for wallet), this section should focus on how wallet contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
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

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the project and ecosystem as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how it fits into the road-map of the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.

# References
[references]: #references

Include any references such as links to other documents or reference implementations.
