
- Title: relative-kernels
- Authors: [Antioch Peverell](mailto:apeverell@protonmail.com)
- Start date: Aug 8, 2019
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Currently we can lock a transaction kernel such that it is only valid after an absolute block height. We can extend this to support relative timelocks by including a reference to a previous transaction kernel by its excess commitment.

# Motivation
[motivation]: #motivation

Absolute timelocks are useful but are susceptible to delay. Any delay between creation and use can have an adverse impact on their usage. An absolute timelock starts counting down as soon as the transaction is built. Relative timelocks can be more useful in some scenarios allowing transactions to be created well in advance of their actual use without impacting the lock period.

Robust atomic swap implementations can be implemented with relative timelocks on the refund transactions, insulating both parties from delays during the swap process, intentional or otherwise.

Similarly, Lightning style payment channels can leverage relative timelocks to implement refunds during non-cooperative payment channel close scenarios.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

Explain the proposal as if it were already introduced into the Grin ecosystem and you were teaching it to another community member. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Grin community members should *think* about the improvement, and how it should impact the way they interact with Grin. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Grin community members and new Grin community members.

For implementation-oriented RFCs (e.g. for wallet), this section should focus on how wallet contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Grin currently supports three kernel variants -

* Plain
* Coinbase
* HeightLocked

These are implemented as kernel "feature" variants -

```
pub enum KernelFeatures {
	/// Plain kernel (the default for Grin txs).
	Plain = 0,
	/// A coinbase kernel.
	Coinbase = 1,
	/// A kernel with an explicit lock height.
	HeightLocked = 2,
}
```

Each kernel variant supports feature specific data -

```
# Plain
{    
  "fee": 8
}

# Coinbase
{
  # empty
}

# Height Locked
{
  "fee": 8,
  "lock_height": 295800
}
```

We propose adding an additional feature variant "Relative Height Locked".

```
pub enum KernelFeatures {
	...
	/// A kernel with a relative lock height.
	RelativeHeightLocked = 3,
}
```

A relative height locked kernel would reference a previous kernel by its excess (commitment)
along with a relative height (height in blocks between the two kernels).

```
# Relative Height Locked
{
  "fee": 8,
  "lock_height": 1044,
  "rel_kernel": "088305235baac64e90daca81b0bad7afbce3a6e49c989572095a892e857e681429"
}
```

In the example above the transaction containing this kernel would only be valid 1044 blocks (approx 24 hours) after the referenced kernel.

While we require a full 32 bytes of data to reference a previous kernel we can optimize this when storing the kernel in the MMR. We append kernels to the MMR based on immutable history and this allows us to reference the prior kernel by MMR position. At a given block height the MMR is immutable and the kernel at a given position is deterministic and immutable. Every node sees a consistent view of kernel ordering in their local copy of the kernel MMR.

```
# Relative Height Locked (MMR storage optimized)
{
  "fee": 8,
  "lock_height": 1044,
  "rel_kernel_pos": 18542
}
```

In this way we can represent relative lock heights in the kernel MMR with an additional 8 bytes for each relative lock height instance.

Every kernel requires 97 bytes for the following -
* 64 bytes for the signature
* 32 bytes for the excess commitment
* 1 byte for the feature variant

A plain kernel requires an additional -
* 8 bytes for the fee

A height locked kernel requires an additional -
* 8 bytes for the fee
* 8 bytes for the lock height

A relative height locked kernel requires an additional -
* 8 bytes for the fee
* 8 bytes for the lock height
* 8 bytes for the referenced kernel MMR position

Kernel sizes are as follows -
* Plain kernel: 97+8 = 105 bytes
* Coinbase kernel: 97 bytes
* Height locked kernel: 97+8+8 = 113 bytes
* Relative height locked kernel: 97+8+8+8 = 121 bytes

Height locked and relative height locked kernels should be relatively rare and these additional bytes should only be rarely required.

Transaction slates (wallet to wallet) and transactions (p2p broadcast and within txpool) will need the full 32 bytes to represent a referenced kernel excess commitment. But once the kernel is appended to the kernel MMR we can save much of this additional overhead and represent the reference to the previous kernel with 8 bytes for the kernel MMR position.

----

*** TODO - Protocol version support. Describe need to bump protocol version. All messages that include kernels (txs, blocks, compact blocks etc.) Also txhashset.zip download...

# Drawbacks
[drawbacks]: #drawbacks

Relative height locked kernels require more bytes to represent. We need an additional 32 bytes when referencing a previous kernel excess commitment. But we can optimize this to 8 bytes to represent the kernel MMR position when storing this additional data in the kernel MMR.

We will need to translate between kernel excess commitments and kernel MMR positions to faciliate this and this will introduce some additional complexity to the implementation.

Relative height locks will introduce some additional complexity around the handling of the txpool, particularly when dealing with forks and chain reorgs. Previously valid transactions may not necessarily be valid after a reorg if they are included in an earlier block for example. We need to consider various edge cases here.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss Bitcoin style timelocks (both for transactions and outputs).
Explain why Grin/MW only has kernels to work with.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Explain why "relative lock heights" against kernel excess commitments is more flexible in some ways (kernels need not exist yet etc).

Kernel excess commitments can be constructed independently from the transactions themselves.
And given kernel offsets we have a lot of flexibility in how multi-party transactions can be built. i.e. parties can cooperatively build a kernel excess first, then build the corresponding transaction, then finally sign the necessary transaction data. This flexibility may open up interesting ways of using relative dependencies between transaction kernels.

Link to the Elder Channels gist.

Think about what the natural extension and evolution of your proposal would be and how it would affect the project and ecosystem as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how it fits into the road-map of the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.

# References
[references]: #references

https://en.bitcoin.it/wiki/Timelock#nLockTime
https://en.bitcoin.it/wiki/Timelock#CheckLockTimeVerify
https://en.bitcoin.it/wiki/Timelock#Relative_locktime
https://en.bitcoin.it/wiki/Timelock#CheckSequenceVerify

Atomic swap docs link.
Elder Channel gist link.
