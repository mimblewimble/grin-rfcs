
- Title: nrd-kernels
- Authors: [Antioch Peverell](mailto:apeverell@protonmail.com)
- Start date: Mar 24, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Grin supports a limited implementation of "relative timelocks" with "No Recent Duplicate" (NRD) transaction kernels. Transactions can be constructed such that they share duplicate kernels. An NRD kernel instance is not valid within a specified number of blocks relative to a prior duplicate instance of the kernel. A minimum number of blocks must therefore exist between two instances of an NRD kernel. This provides a relative timelock between transactions.

# Motivation
[motivation]: #motivation

Relative timelocks are a prerequisite for robust payment channels. NRD kernels can be used to implement a _revocable_ channel close mechanism.
A mandatory revocation period can be introduced through a relative timelock between two transactions. Any attempt to close an old invalid channel state can be safely revoked during the revocation period.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

A minimum distance in block height is enforced between successive duplicate instances of a given NRD kernel. This can be used to enforce a relative lock height between two transactions. A transaction containing an NRD kernel will not be accepted as valid within the specified block height relative to any prior instance of the NRD kernel.

Transactions can be constructed around an existing transaction kernel by introducing either an additional kernel or in some cases by simply adjusting the kernel offset. This allows NRD kernels to be used across any pair of transactions.

The NRD kernel implementation prioritizes simplicity and minimalism. Grin does not support a general solution for arbitrary locks between arbitrary pairs of kernels. The implementation is restrictive for reasons of performance and long term scalability. References between duplicate kernels are _implicit_, avoiding the need to store kernel references. Locks are limited in length to _recent_ history, avoiding the need to inspect the full historical kernel set.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.


----

An NRD kernel is not valid within a specified number of blocks of a previous duplicate instance of the same NRD kernel. We define duplicate here as two NRD kernels sharing the same public excess commitment. NRD kernels with different excess commitments are not treated as duplicates. An NRD kernel and a non-NRD kernel (plain kernel, coinbase kernel etc.) sharing the same excess commitment are not treated as duplicates.

An NRD kernel has an associated _relative_ lock height. For a block containing this kernel to be valid, no duplicate instance of the kernel can exist in any previous block closer within this relative height.
For example a transaction containing an NRD kernel with relative lock height 1440 (approx 24 hours) is included in a block at height 1000000. This block is only valid if no duplicate instance of this kernel exists in any block from height 998560 (h-1440) to height 999999 (h-1).
If no duplicate instance of the kernel exists within this range then the lock criteria is met. If a duplicate exists outside of this range, earlier than block 998,560 then the lock criterai is still met and the block is valid. Thus the lock defaults to "fail open" and only recent history need be looked at. A kernel can be delayed by the existence of a previous kernel. The _non-existence_ of a previous kernel has no impact on the lock criteria. Note that this implies the _first_ occurrence of any NRD kernel meets the lock criteria trivially.

Each node maintains an index of _recent_ NRD kernels to enable efficient checking of NRD relative lock heights. Note we only need to index NRD locks and we only need to index those within recent history. Relative locks longer than 7 days are not valid. This is believed to be sufficient to cover all proposed use cases.

The minimum value for a relative lock height is 1 meaning a prior instance of the kernel can exist in the previous block for the lock criteria to be met.
An instance of the NRD kernel in the _same_ block will invalidate the block as the lock criteria will not be met.

NRD lock heights of 0 are invalid and it is never valid for two duplicate instances of the _same_ NRD kernel to exist in the same block.

It follows that two transactions contaning duplicate instances of the same NRD kernel cannot be accepted as valid in the txpool concurrently.
"First one wins" semantics apply when validating transactions containing NRD kernels in a similar way to resolving the spending of unspent outputs.

Grin supports "rewind" back through recent history to handle fork and chain reorg scenarios. 1 week of full blocks are maintained on each node and up to 10080 blocks can be rewound. To support relative lockheights each node must maintain an index over sufficient kernel history for an _additional_ 10080 blocks beyond this rewind horizon. Each node should maintain 2 weeks of kernel history in the local NRD kernel index. This will cover the pathological case of a 1 week rewind and the validaton of a 1 week long relative lock beyond that. The primary use case is for revocable payment channel close operations. We believe a 7 day period is more than sufficient for this. We do not require long, extended revocation periods and limiting this to a few days is preferable to keep the cost of verification low. The need for these revocable transactions to be included on chain should be low as these are only required in a non-cooperative situation but where required we want to minimize the cost of verification which must be performed across all nodes.

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

The maximum supported NRD _relative_height_ is 10080 (7 days) and the relative height can be safely and conveniently represented as a `u16` (2 bytes). This differs from absolute lock heights where `u64` (8 bytes) is necessary to specify the lock height.

The minimum supported NRD _relative_height_ is 1 and a value of 0 is not valid. Two duplicate instances of a given NRD kernel cannot exist simultaneously in the same block. There must be a relative height of at least 1 block between them.

----

Nodes on the Grin network currently support two serialization versions for transaction kernels -

__V1 "fixed size kernels"__

In V1 all kernels are serialized to the same "fixed" number of bytes.
Every kernel includes 8 bytes for the fee with a 0 fee if not applicable.
Every kernel also includes 8 bytes of feature specific data. This is used for the lock height and unused for all other kernel variants.

* kernel features
	* feature byte: 1 byte
	* fee: 8 bytes
	* feature specific data: 8 bytes
* excess commitment: 33 bytes
* signature: 64 bytes

V1 is supported for backward compatibility with older nodes and will be used as necessary, based on version negotiation during the peer connection setup process.

__V2 "variable size kernels"__

V2 kernels have been supported since Grin `v2.1.0` and V2 supports the notion of "variable size" kernels.

In V2 we only serialize the data relevant for each kernel feature variant. Each kernel contains a 33 byte excess commitment and associated 64 byte signature. But the feature variant now defines the additional data.

* Plain kernel:
	* feature: 1 byte
	* fee: 8 bytes
* Coinbase kernel:
	* feature: 1 byte
* Height locked kernel:
	* feature: 1 byte
	* fee: 8 bytes
	* lock height: 8 bytes
* NRD kernel:
	* feature: 1 byte
	* fee: 8 bytes
	* relative lock height: 2 bytes

When taking advantage of variable size kernels, NRD kernels are only 2 bytes larger than plain kernels to account for the additional relative lock height information.

Note that the serialization strategy is used for both network "on the wire" serialization of both transactions and full blocks, and local storage, both the database for full blocks and the kernel MMR backend files.
Version negotiation occurs during the initial peer connection setup process and determines which version is used for p2p message serialization.
Note that if a node uses V2 serialization for the kernel MMR backend file then it will provide a V2 txhashset based on these underlying files.

----

No additional data is introduced with NRD kernels. There is no opportunity to include arbitrary data. Any additional kernel included in a transaction is itself still a fully valid kernel. There is no explicit reference necessary that could be misused to include arbitrary data.

An additional NRD kernel in a transaction will increase the "weight" of the transaction by this single additonal kernel and allows for a simple way to deal with additional fees. A transaction with an additional kernel must provide additional fees to cover the additional "weight". NRD kernels cannot be added for free. Note that in some limited situations it is possible to _replace_ a kernel with an NRD kernel. If the NRD lock can be introduced without adding an additional kernel then the fee does not have to be increased and the lock is effectively added for free.

----

A transaction kernel consists of an excess commitment and an associated signature showing this excess is indeed a commitment to 0.

A transaction with a single kernel can always be represented as a transaction with multiple kernels, provided the kernels excess commitments sum to the correct total excess.

Given an existing NRD kernel with excess commitment -

* _r'G + 0H_

And a transaction with single excess commitment -

* _rG + 0H_

This transaction can be represented as a pair of kernels with excess commitments -

* _rG + 0H =
    (r'G + 0H) + (r-r'G + 0H)_

We take advantage of this to allow an arbitrary NRD kernel to be included in any transaction at construction time.

Additionally the kernel offset incuded in each transaction can be used in certain situations to allow the replacement of a single transaction kernel with an NRD kernel without needing to introduce an additional kernel.

Given an existing NRD kernel with excess commitment -

* _r'G + 0H_

And a transaction with single excess commitment and kernel offset -

* _rG + 0H, o_

This transaction can be rewritten to use the NRD kernel -

* _r'G + 0H, (o+r-r')_

These two "degrees of freedom", introducing multiple kernels and adjusting the kernel offset, allowing for flexibility to introduce an NRD kernel in a variety of ways.

* Introduce NRD kernel to transaction, compensate with additional kernel.
* Introduce NRD kernel to transaction, compensate with kernel offset.

----

# Drawbacks
[drawbacks]: #drawbacks

NRD kernels are a limited and restricted form of "relative locks" between kernels. These locks are limited to a period of 7 days and "fail open" beyond that window. This approach meets the requirements for limited revocable payment channel operations but there are likely to be use cases where this approach is not sufficient or unsuitable.

While it would be nice to provide a fully general purpose solution that would allow arbitrary locks to be implemented, it does appear to be hard, if not impossible, to do this in Grin/MW.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Referencing historical data in Grin and in mimblewimble in general is difficult due to the possibility of pruning historical data. It is not possible to reference old outputs once they are spent. Historical validators must have access to any referenced data to validate consensus rules. This leaves transaction kernels as the only available data to be referenced. While arbitrary historical kernels _can_ be referenced this is not desirable as we do not want to impose additional constraints on nodes, requiring them to maintain historical data that would otherwise be prunable.

An earlier design iteration was "No Such Kernel Recently" (NSKR) locks. Where NRD references were implicit, with duplicate kernel excess commitments, NSKR kernels referenced prior kernels explicitly. These explicit references were problematic for several reasons -

* Additional overhead, both local storage and network traffic due to the explicit references
* Optimization by referencing prior kernel based on MMR position introduced external data, making kernels harder to validate in soliation.
* Permitting non-existence of references due to limited window of history, opened up a vector for "spam" where arbitrary data could be used in place of a valid reference.

To prevent "spam" it was observed that we could use a signature to verify the reference was indeed a valid reference to a prior kernel excess commitment. This effectively made a reference a copy of a full kernel, with excess commitment and associated signature. At this point it was observed that "duplicate" kernels might solve the problem in an elegant way.

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

Some investigation is still needed around the conditions necessary to allow a kernel to simply be reused with an adjustment to the kernel offset and where an additional kernel is necessary. An adjustment to the kernel offset will expose the private excess under certain conditions and cannot be done safely for all transactions.

One outstanding question is whether NRD kernels are sufficient to cover all "relative timelock" requirements. We believe them to be sufficient for the revocable payment channel close mechanism. But they may not be sufficient for all use cases.

# References
[references]: #references

[link to original "trigger" post on mailinglist]
[link to NSKR writeup]
[link to NSKR based Elder Channel writeup]
[link to relative locks in bitcoin]
