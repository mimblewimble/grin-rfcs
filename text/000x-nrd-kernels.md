
- Title: nrd-kernels
- Authors: [Antioch Peverell](mailto:apeverell@protonmail.com)
- Start date: Mar 24, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Grin supports a limited implementation of "relative timelocks" with the "No Recent Duplicate" (NRD) transaction kernel variant. Transactions can be constructed such that kernels are "reused" and duplicated. An NRD kernel instance is not valid within a specified number of blocks relative to a prior instance of the same (duplicate) kernel. A minimum number of blocks must therefore exist between two instances of an NRD kernel. This provides a relative timelock between transactions.

# Motivation
[motivation]: #motivation

Relative timelocks are a prerequisite for robust payment channels. NRD kernels can be used to implement a _revocable_ channel close mechanism.
A mandatory revocation period can be introduced through a relative timelock between two transactions. Any attempt to close an old invalid channel state can be safely revoked during the revocation period.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

A minimum distance in block height is enforced between successive duplicate instances of a given NRD kernel. This can be used to enforce a relative lock height between two transactions. A transaction containing an NRD kernel will not be accepted as valid within the specified block height relative to any prior instance of the NRD kernel.

Transactions can be constructed around an existing transaction kernel by introducing either an additional kernel or in some cases by simply adjusting the kernel offset. This allows NRD kernels to be used across any pair of transactions.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.


----

An NRD kernel is not valid within a specified number of blocks of a previous duplicate instance of the same NRD kernel. We define duplicate here as two NRD kernels sharing the same public excess commitment. NRD kernels with different excess commitments are not treated as duplicates. An NRD kernel and a non-NRD kernel (plain kernel, coinbase kernel etc.) sharing the same excess commitment are not treated as duplicates.

[rewrite below from perspective or later kernel, not first kernel]
A "No Recent Duplicate" kernel has an associated _relative_ lock height. Once this kernel is included in a block on-chain the timelock starts.
Another instance of the _same_ kernel will not be accepted as valid until the specified number of blocks has elapsed.
For example if an NRD kernel is accepted at height 500,000 and specifies a relative lock height of 1,440 (24 hours) then a subsequent instance of the kernel will not be accepted as valid until block height 501,440.
One instance of an NRD kernel can be said to slow a subsequent instance down by delaying it from being accepted and included in a block.

Each node maintains an index of recent NRD kernels to enable efficient validation of this rule.

NRD lock heights are limited to a maximum of 10,080 (7 days). This limits the size of the window of recent kernel activity that must be indexed on each node.
Locks cannot be created larger than 7 days but this is believed to cover all proposed use cases currently.

The minimum value for a relative lock height is 1 meaning the subsequent kernel instance is valid in the _next_ block.
NRD lock heights of 0 are not valid and it is never valid for two duplicate instances of the _same_ NRD kernel to exist in the same block.
This implies that two transactions contaning duplicate instances of the same NRD kernel will not be accepted as valid concurrently in the txpool.
"First one wins" semantics apply when validating transactions containing NRD kernels in a similar way to resolving double spends of unspent outputs.

NRD kernels are similar to absolute height locked kernels in that both kernel variants specify a lock height. But they differ in one important aspect; an instance of an absolute height locked kernel is _itself_ not valid until the specified block height, whereas the presence of an NRD kernel on-chain will delay the _subsequent_ instance of that kernel.





----

The following kernel variants are supported in Grin -

* Plain
* Coinbase
* HeightLocked
* NoRecentDuplicate

These are implemented as kernel "feature" variants -

```
pub enum KernelFeatures {
	/// Plain kernel (default for Grin txs).
	Plain = 0,
	/// A coinbase kernel.
	Coinbase = 1,
	/// A kernel with an explicit absolute lock height.
	HeightLocked = 2,
	/// A relative height locked NRD kernel.
	NoRecentDuplicate = 3,
}
```

Each kernel variant includes feature specific data -

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
# No Recent Duplicate (NRD)
{
  "fee": 8,
  "relative_height": 1440,
}
```

Note that NRD kernels require no additional data beyond that required for absolute height locked kernels. The reference to the previous kernel is _implicit_ and based on a duplicate kernel excess commitment.

The maximum supported NRD _relative_height_ is 10,080 (7 days) and can be safely represented as a `u16`.

The minimum supported NRD _relative_height_ is 1 and a value of 0 is not valid. Two duplicate instances of a given NRD kernel cannot exist simultaneously in the same block. There must be a relative height of at least 1 block between them.

----

[fail open]



----

----

No additional data is introduced with NRD kernels. There is no opportunity to include spam or arbitrary data on the block chain. Any additional transaction kernel is still a full transaction kernel. The excess commitment is simply a commitment to 0 and the associated signature verifies this.

An additional NRD kernel in a transaction will increase the "weight" of the transaction by this single additonal kernel and allows for a simple way to deal with additional fees. A transaction with an additional kernel must provide additional fees to cover the additional "weight". NRD kernels cannot be added for free. Note that in some limited situations it is possible to _replace_ a kernel with an NRD kernel. If the NRD lock can be introduced without adding an additional kernel then the fee does not have to be increased and the lock is effectively added for free.

----


----
A transaction kernel consists of an excess commitment and an associated signature showing this excess is indeed a commitment to 0.

A transaction with a single kernel can always be represented as a transaction with multiple kernels, provided the kernels excess commitments sum to the correct total excess.

Given an existing NRD kernel with excess commitment -

* _r'G + 0H_

And a transaction with single excess commitment -

* _rG + 0H_

This transaction can be rewritten with a pair of kernels with excess commitments -

* _rG + 0H = (r'G + 0H) + (r-r'G + 0H)_

We take advantage of this to allow an arbitrary NRD kernel to be included in any transaction at construction time.

Additionally the kernel offset incuded in each transaction can be used in certain situations to allow the replacement of a single transaction kernel with an NRD kernel without needing to introduce an additional kernel.

Given an existing NRD kernel with excess commitment -

* _r'G + 0H_

And a transaction with single excess commitment and kernel offset -

* _rG + 0H, o_

This transaction can be rewritten to use the NRD kernel -

* _r'G + 0H, o+r-r'_

These two "degrees of freedom", introducing multiple kernels and adjusting the kernel offset, allowing for flexibility to introduce an NRD kernel in a variety of ways.

* Introduce NRD kernel to transaction, compensating with additional kernel.
* Introduce NRD kernel to transaction, compensating with kernel offset.

----



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
